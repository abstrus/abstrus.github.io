---
layout: post
title:  Curing Domain Model Anemia - Part 2
---

In the quest for a rich domain model, we are struggling to have entities injected with services.
In the previous post on the subject, we defined what we thought would be nice to have and ran
into some trouble with the persistence layer.  This time, we'll address one of those issues :
injecting dependencies into our entity repositories.

I keep code samples [available on github](https://github.com/abstrus/AbstrusRichModelBundle). 
**Don't expect that code to work out of the box yet.**  Do not *copy-waste* it like an idiot !

## Injecting Services in a Doctrine Repository

Most often, entities are retrieved from repositories and repositories are retrieved from the entity
manager.  In our case, the repository will certainly needs to be injected with dependencies.  This 
is not a problem if you define your repositories as services and use them as regular services 
provided by the dependency injection container (using setter injection or a decorator, as described 
[here by Jurian Sluiman](https://juriansluiman.nl/article/142/dependency-injection-in-a-doctrine-repository)).
On the other hand, we may want to retrieve the repository using the entity manager for backward 
compatibility reasons.  The solution provided above by Sluiman is not well suited for a this.  We'll
try to address this.

Let's say we want to use our newly rich domain object in the usual fashion as shown in most
documentations.

{% highlight php startinline %}
// AcmeBundle/Controller/SomeController.php
public function showAction($companyId)
{
    $company = $this->getDoctrine()->getRepository('AcmeBundle:Company')
        ->find($companyId);
    
    return $company;
}
{% endhighlight %}

That `getRepository` method is provided by the entity manager.  The solution will be different 
whether the version of Doctrine ORM we use is older than 2.4 or not.  If it is, I would suggest to 
update.  If it is not possible, you will have to use a custom `EntityManager` and reimplement the 
`getRepository` so it use some factory to provide repositories.  Else, Doctrine added that feature 
for us as stated in the 
[2.4 release blog post](http://www.doctrine-project.org/2013/09/11/doctrine-2-4-released.html). 
Also, DoctrineBundle 
[added that configuration option](https://github.com/doctrine/DoctrineBundle/pull/204).  Let's have 
a closer look at that repository factory.  We need to implement 
[`RepositoryFactory`](http://www.doctrine-project.org/api/orm/2.4/class-Doctrine.ORM.Repository.RepositoryFactory.html)
which defines only the method `getRepository`;

{% highlight php startinline %}
// AcmeBundle/ORM/RepositoryFactory.php
use Doctrine\ORM\Repository\RepositoryFactory,
    Doctrine\ORM\EntityManagerInterface,
    Doctrine\ORM\ObjectRepository;

class DIAwareRepositoryFactory
    implements RepositoryFactory
{
    protected $repositoryServices = [];
    
    protected $defaultFactory;
    
    public function __construct(RepositoryFactory $default)
    {
        $this->defaultFactory = $default;
    }
    
    public function getRepository(EntityManagerInterface $entityManager, $entityName)
    {
        if (isset($this->repositoryServices[$entityName])) {
            return $this->repositoryServices[$entityName];
        }
        
        return $this->defaultFactory->getRepository($entityManager, $entityName);
    }
    
    public function subscribeRepository($relatedEntityName, ObjectRepository $repository)
    {
        $this-repositoryServices[$relatedEntityName] = $repository;
    }
}   
{% endhighlight %}

Define this factory as a regular service (I'll suppose it is called `repository_factory`) and tell
DoctrineBundle to use this one instead of the default one.  Don't forget to pass an instance of
`DefaultRepositoryFactory` as a mandatory dependency to our custom factory so it can handle 
not-custom calls.  Next, we define a compiler pass that calls `subscribeRepository` on our 
`DIAwareRepositoryFactory` with any custom repository we defined as service using a tag.

{% highlight php startinline %}
// AcmeBundle/DependencyInjection/Compiler/CustomRepositoryPass.php

use Symfony\Component\DependencyInjection\ContainerBuilder,
    Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface,
    Symfony\Component\DependencyInjection\Reference;

class CustomRepositoryPass implements CompilerPassInterface
{
    const TAG = 'custom_repository';
    const FACTORY_ID = 'repository_factory';

    public function process(ContainerBuilder $container)
    {
        if (!$container->hasDefinition(static::FACTORY_ID)) {
            return;
        }

        $repositoryFactory = $container->getDefinition(static::FACTORY_ID);

        foreach ($container->findTaggedServiceIds(static::TAG) as $repositoryId => $tags) {
            foreach ($tags as $tag) {
                $entityName = $tag['entity_name'];
                $repositoryFactory->addMethodCall(
                    'subscribeRepository',
                    array($entityName, new Reference($repositoryId))
                );
            }
        }
    }
}
{% endhighlight %}

And define your custom repositories injected with services like this:

{% highlight yaml startinline %}
# AcmeBundle/Resources/config/services.yml
services:
    acme.company_repository:
        class: 'Acme\Model\Entity\Repository\CompanyRepository'
        agruments:
            entityManager:  '@doctrine.orm.entity_manager'
            entityClass:    'Acme\Model\Entity\CompanyValue'
            humanResources: '@acme.humanresources_service'
        tags:
            -
                name: 'custom_repository'
                entity_name: 'Acme\Model\Entity\Company"'
{% endhighlight %}

For more details about service definition, using tags and about compiler passes, read some of the
[Symfony Book](http://symfony.com/doc/current/book/index.html) and the 
[Symfony Cookbook](http://symfony.com/doc/current/cookbook/index.html).

With this setup the custom repository should be accessible via 
`$this->getDoctrine()->getRepository('Acme\Model\Entity\Company')` as stated in the controller 
listing.