---
layout: post
title:  Injection de dépendance avec Doctrine - Deuxième partie
categories: fr
---

*Je ne traduirai plus les exemples de code jugeant que ceci est inutile étant donné que les 
mots-clef doivent rester en anglais de toute façon.*

Dans notre quête de modélisation riche d'un domaine d'affaire, nous tentons d'injecter des
dépendances à nos entités Doctrine.  Dans l'article précédent, nous avons déterminé ce qu'il
nous faut et avons eu certaines difficultés avec la couche d'accès aux données persistantes.
Nous avons vu qu'en empruntant une idée du C++, nous allions pouvoir contourner ce détail. Par
contre, nous aurons besoin d'injecter des services dans nos dépôts d'entités.  C'est ce que nous
détaillerons ici.

In the quest for a rich domain model, we are struggling to have entities injected with services.
In the previous post, we defined what we thought would be nice to have and ran
into some troubles with the persistence layer.  This time, we'll address a specific issue :
injecting dependencies into our entity repositories.

Il y a des exemples de codes plus charnus [sur Github][github-richmodel]{:target="_blank"}. 
**Ne supposez pas que le code fonctionnera directement.**  Ce ne sont que des exemples et des 
brouillons.

## Injection de services dans un dépôt Doctrine

Normalement, les entités sont récupérée des dépôts et les dépôts sont fournis par le gestionnaire
d'entités.  Il n'y a pas de problème à déclarer un dépôt en tant que service, y injecter toutes
sortes de dépendances et l'utiliser via le conteneur Symfony en utilisant l'injection par méthode
ou un décorateur, tel que décrit par Jurian Sluiman dans 
[son blog][sluiman-didoctrine]{:target="_blank"}. D'un autre côté, si partout dans le reste de 
l'application nous utilisons le gestionnaire d'entités pour récupérer les dépôts, la solution de 
Sluiman ne sera pas suffisante.

Supposons que nous voulons récupérer un objet métier de manière usuelle comme suit.

{% highlight php startinline %}
// AcmeBundle/Controller/SomeController.php
public function showAction($companyId)
{
    $company = $this->getDoctrine()->getRepository('AcmeBundle:Company')
        ->find($companyId);
    
    return $company;
}
{% endhighlight %}

Si la fonction `Company::__construct` n'a pas exactement la même signature que celle de ça 
[classe mère][api-entityrepository]{:target="_blank"}, ce code plantera.

La solution dépend en partie de la version de Doctrine que vous utilisez.  Si vous utilisez une
version antérieure à 2.4, je recommande de mettre à jour.  Si ce n'est pas possible, vous devrez
faire une implémentation maison de la classe `EntityManager` et redéfinir la méthode `getRepository`
pour qu'elle utilise une usine à dépôt sur mesure.  Sinon, on a ajouté cette fonctionnalité à
Doctrine tel que mentionné dans l'[annonce de la sortie 2.4][doctrine-release2.4]{:target="_blank"}. 
De plus, le bundle DoctrineBundle comporte une 
[option de configuration][github-doctrinebundle-repositoryfactory]{:target="_blank"}  à cet effet.
Regardons de plus près cette usine à dépôt. Nous aurons besoin d'implémenter l'interface 
[`RepositoryFactory`][api-repositoryfactory]{:target="_blank"}.

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
        // Essaie de fournir un dépôt maison
        if (isset($this->repositoryServices[$entityName])) {
            return $this->repositoryServices[$entityName];
        }
        
        // sinon, adopte le comportement par défaut.
        return $this->defaultFactory->getRepository($entityManager, $entityName);
    }
    
    public function subscribeRepository($relatedEntityName, ObjectRepository $repository)
    {
        $this->repositoryServices[$relatedEntityName] = $repository;
    }
}   
{% endhighlight %}

Vous pouvez ensuite déclarer cette usine comme un service normal (supposons qu'il s'appelle 
`repository_factory`) et indiquer à la configuration de DoctrineBundle d'utiliser celui-ci à
la place.  Il ne faut pas oublier de lui passer une instance de `DefaultRepositoryFactory`
pour assurer une rétro-compatibilité. 

Nous devons ensuite coder une passe de compilation (*compiler pass*) pour indiquer à notre
`DIAwareRepositoryFactory` quels sont les dépôt sur mesure que nous voulons qui prenne en charge.

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

Finalement, nous définissons nos dépôts comme des services à envoyer à notre usine comme ceci :

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

Pour plus de détails sur la définition de services, l'utilisation d'étiquettes ou sur les passes
de compilation, lisez les livres [Symfony Book][symfonybook]{:target="_blank"} et [Symfony Cookbook][symfonycookbook]{:target="_blank"}.

Avec toute cette mise en place, notre dépôt d'entité personnalisé devrait être accessible via 
`$this->getDoctrine()->getRepository('Acme\Model\Entity\Company')` tel que montré dans l'exemple
de contrôleur.  Nous pouvons maintenant utiliser des dépôts munis de dépendances arbitraires. C'est
un bon départ !  La manière de récupérer des instance de `Company` n'est pas encore évidente.  Nous
détaillerons ce point dans un prochain article.


[api-entityrepository]:   http://www.doctrine-project.org/api/orm/2.4/source-class-Doctrine.ORM.EntityRepository.html#___construct  "EntityRepository API Documentation"
[api-repositoryfactory]:  http://www.doctrine-project.org/api/orm/2.4/class-Doctrine.ORM.Repository.RepositoryFactory.html          "RepositoryFactory API Documentation"

[symfonybook]:            http://symfony.com/doc/current/book/index.html                                                            "Symfony Online Book"
[symfonycookbook]:        http://symfony.com/doc/current/cookbook/index.html                                                        "Symfony Online Cookbook"

[github-richmodel]:       https://github.com/abstrus/AbstrusRichModelBundle                                                         "AbstrusRichModelBundle on Github"

[github-doctrinebundle-repositoryfactory]: https://github.com/doctrine/DoctrineBundle/pull/204 "RepositoryFactory configuration - Github Pull Request"

[doctrine-release2.4]: http://www.doctrine-project.org/2013/09/11/doctrine-2-4-released.html "Doctrine 2.4 Released - Doctrine blog"

[sluiman-didoctrine]: https://juriansluiman.nl/article/142/dependency-injection-in-a-doctrine-repository "Jurian Sluiman about Dependency Injection with Doctrine repositories"