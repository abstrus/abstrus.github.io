---
layout: post
title:  Dependency injection with Doctrine - Part 3
translation: fr
---

On [part 2][abstruscodes-previous], we managed to have dependencies injected in Doctrine 
repositories.  This time, we would like to have dependencies injected in Doctrine entities.
There are many ways to achieve this. 

I keep code samples [available on github][github-richmodel]{:target="_blank"}. 
**Don't expect that code to work out of the box yet.**  Do not *copy-waste* it like an idiot !

## Half Repository, Half Factory

Our custom repository need to return `Company` objects, not `CompanyValue` objects.
We could simply do the following with any `find*` method:

{% highlight php startinline %}
// AcmeBundle/Model/Entity/Repository/CompanyRepository
class CompanyRepository
    extends EntityRepository
{
    protected $humanResourcesService;
    
    public function __construct(
         Doctrine\ORM\EntityManager $entityManager, 
         $entityClass,
         $humanResources
    )
    {
        $metadata  = $entityManager->getClassMetadata($entityName);
        
        parent::__construct($entityManager, $metadata);
        
        $this->humanResourcesService = $humanResources;
    }

    // ... other methods

    public function find($companyId)
    {
        return new Company(parent::find($companyId), $this->humanResourcesService);
    }
}
{% endhighlight %}

This is not a very good solution for many reasons, one of them being that we cannot handle results from
QueryBuilder and Query instances.  Repositories often provides instances of those two classes in case
there is need to paginate the results.  

This part may be a bit tricky.  It really depends on how the entity repository is intended to be used.
As it provides way more flexibility and power without adding much complexity, I would recommend to
**never use** the `find*` magic methods and use the QueryBuilder and Query objects instead.  This 
[post][eberlei-repositories]{:target="_blank"} from Benjamin Eberlei provides a really nice start 
for using a variant of the [specification pattern][wiki-specification]{:target="_blank"} with 
Doctrine.  More so, the data hydrated using the magic methods do not use the same work flow as with 
DQL.  I am far from being a Doctrine-guru, but I don't like it when there are many ways of doing the 
same thing.

Back to our business, we need to hydrate a Company object with services and a CompanyValue.  To 
hydrate things, we need a `Hydrator`.  

## Right Hydration for a Thirsty Business Model

If you look down in Doctrine's internals, you should find some 
[hydrators][api-hydration]{:target="_blank"}. A hydrator transforms a SQL result set in something 
else.  Doctrine provides an [`ObjectHydrator`][api-objecthydrator]{:target="_blank"}
but it cannot directly let us inject dependencies in hydrated entities.

Hydrators are provided by the entity manager using the `newHydrator($hydrationMode)` method where 
the parameter is the chosen hydrator's identifier.  Unlike the repository factory which was made
customizable since Doctrine 2.4, if we want the entity manager to provide hydrators using dependency
injection, we will have to make a custom entity manager and overwrite the `newHydrator` method.

{% highlight php startinline %}
<?php
// AcmeBundle/ORM/EntityManager.php

use Doctrine\ORM\EntityManager as DoctrineEntityManager;
    Doctrine\ORM\Internal\Hydration\AbstractHydrator;

class EntityManager 
    extends 
        DoctrineEntityManager
{
    protected $hydratorServices = array();
    
    public function newHydrator($hydrationMode)
    {
        if (isset($this->hydratorServices[$hydrationMode])) {
            return $this->hydratorServices[$hydrationMode];
        }
        
        return parent::newHydrator($hydrationMode);
    }
    
    public function subscribeHydrator($hydrationMode, AbstractHydrator $hydrator)
    {
        $this->hydratorServices[$hydrationMode] = $hydrator;
    }
}
{% endhighlight %}

The `subscribeHydrator` method will be used by a compiler pass.  We adopt exactly the same
strategy we used with the repository factory and the `subscribeRepository` method.  There is a
[draft compiler pass implementation][github-customhydrationcompilerpass]{:target="_blank"}
for this purpose on the dedicated Github repository

The hydrators themselves are up to you.  I have though of a possible implementation that uses a
dedicated factory for creating rich domain entities.  I would recommend to extends the
`ObjectHydrator`, overwrite the `hydrateAll` and `hydrateRow` methods and use their respective
`parent::` implementations to get the persistent data.  You can find an 
[example implementation][github-domainobjecthydrator]{:target="_blank"} of this hydrator in the 
usual Github repository along with the associated [factory interface][github-entityfactory]{:target="_blank"}
and an [implementation][github-fromdraftfactory]{:target="_blank"}
based on [object cloning][api-phpcloning]{:target="_blank"} mechanisms.

## Tools of the trade

Having the dependency injection wired with Doctrine's repositories, entities (somewhat) and 
hydrators should help building a rich domain model in many projects.  Remember the `Company` class 
example we used in [the first post of this trilogy][abstruscodes-first]{:target="_blank"} ?  We now
have the proof that entities returned by Doctrine queries can order pizza. 
[QED][wiki-qed]{:target="_blank"} as the mathematicians say.



[abstruscodes-previous]:      {{ site.url }}dependency-injection-with-doctrine-part-2.html "Previous part"
[abstruscodes-first]:         {{ site.url }}dependency-injection-with-doctrine-part-1.html "First part"

[github-richmodel]: https://github.com/abstrus/AbstrusRichModelBundle "AbstrusRichModelBundle - Github"
[github-domainobjecthydrator]: https://github.com/abstrus/AbstrusRichModelBundle/blob/master/ORM/DomainObjectHydrator.php "DomainObjectHydrator - Github"
[github-entityfactory]: https://github.com/abstrus/AbstrusRichModelBundle/blob/master/Model/Entity/Factory/EntityFactory.php "EntityFactory - Github"
[github-fromdraftfactory]: https://github.com/abstrus/AbstrusRichModelBundle/blob/master/Model/Entity/Factory/FromDraftFactory.php "FromDraftFactory - Github"
[github-customhydrationcompilerpass]: https://github.com/abstrus/AbstrusRichModelBundle/blob/master/DependencyInjection/Compiler/CustomHydratorCompilerPass.php "CustomHydratorCompilerPass - Github"

[api-phpcloning]:     http://www.php.net/manual/en/language.oop5.cloning.phpcloning  "Cloning Objects - PHP Documentation"
[api-hydration]:      http://www.doctrine-project.org/api/orm/2.4/namespace-Doctrine.ORM.Internal.Hydration.html  "Hydration API Documentation"
[api-objecthydrator]: http://www.doctrine-project.org/api/orm/2.4/class-Doctrine.ORM.Internal.Hydration.ObjectHydrator.html "ObjectHydrator API Documentation"

[wiki-specification]: http://en.wikipedia.org/wiki/Specification_pattern "Specification Pattern - Wikipedia"
[wiki-qed]:           http://en.wiktionary.org/wiki/quod_erat_demonstrandum "Quod erat demonstrandum - Wikipedia"

[eberlei-repositories]: http://whitewashing.de/2013/03/04/doctrine_repositories.html "Taming Doctrine Repositories - Whitewashing Blog"