---
layout: post
title:  Injection de dépendance avec Doctrine - Troisième partie
categories: fr
---

Dans l'[article précédent][abstruscodes-previous],  nous avons réussi à utiliser l'injection de 
dépendances avec les dépôts Doctrine.  Cette fois, nous voudrions utiliser l'injection de dépendances
dans les entités Doctrine.  Il y a plusieurs approches possibles.

repositories.  This time, we would like to have dependencies injected in Doctrine entities.
There are many ways to achieve this. 

Il y a des exemples de codes plus charnus [sur Github][github-richmodel]{:target="_blank"}. 
**Ne supposez pas que le code fonctionnera directement.**  Ce ne sont que des exemples et des 
brouillons.

## Moitié dépôt, moitié usine

Nous voulons que les dépôts produisent des instances de `Company` et non de `CompanyValue`.  Nous
pourrions implémenter les méthodes `find*` de la manière suivante:

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

Ce n'est pas optimal pour plusieurs raisons, une étant que nous ne pouvons pas ainsi prendre en 
charge les résultats produits pas les assembleurs de requêtes de Doctrine (*QueryBuilder* et *Query*).
De tels objets sont fréquemment utilisés notamment pour implémenter une pagination des résultats.

Cette partie est délicate et fonction de la manière avec laquelle les dépôts d'entités seront utilisés.
Pour des raisons de flexibilité, je recommande de systématiquement utiliser les assembleurs
de requête et d'éviter le plus possibles les méthodes magiques de la famille de `findBy`. 
[Cet article][eberlei-repositories]{:target="_blank"} de Benjamin Eberlei propose de très bonnes
pistes de solutions pour utiliser une variante du patron de conception 
[*specification*][wiki-specification]{:target="_blank"} avec Doctrine.

## La bonne hydratation pour un modèle d'affaire assoiffé

Plus profondément dans les composants internes de Doctrine, on trouve les 
[hydrateurs][api-hydration]{:target="_blank"}.  Ces objets transforment les résultats des requêtes
SQL en d'autres choses, comme des entités (objets), par exemple.  Doctrine fourni le 
[`ObjectHydrator`][api-objecthydrator]{:target="_blank"}, mais il ne peut pas nous servir 
directement pour injecter des dépendances dans les entités nouvellement récupérées.

Les hydrateurs sont eux aussi fournis par le gestionnaire d'entités, en utilisant la méthode
`newHydrator($hydrationMode)`.  *a contrario* de l'usine à dépôt qui est devenue configurable avec
Doctrine 2.4, nous n'avons pas le choix de redéfinir le gestionnaire d'entité.

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

La méthode `subscribeHydrator` sera utilisée par une passe de compilation de manière analogue à
l'usine à dépôt que nous avons implémenté au précédent article.  Il y a une ébauche d'une telle 
passe sur le [dépôt Github][github-customhydrationcompilerpass]{:target="_blank"}.

Il vous reste encore à implémenter vos hydrateurs.  C'est à vous de voir la meilleure manière de
faire.  Je recommande d'hériter du `ObjectHydrator` et de redéfinir les méthodes `hydrateAll` et
`hydrateRow`.  Vous pouvez trouver un [exemple][github-domainobjecthydrator]{:target="_blank"} sur 
le dépôt Github.  Vous y trouverez aussi une idée d'[usine][github-fromdraftfactory]{:target="_blank"} 
basée sur le [clonage d'objets][api-phpcloning]{:target="_blank"}.

## Bien outillé pour le métier

Pouvoir utiliser l'injection de dépendances dans toutes ces composantes de Doctrine devrait
faciliter la construction de modèles métiers riches dans plusieurs projets.  Vous-souvenez vous de
la classe `Compagnie` que nous avions utilisée dans le 
[premier article de la série][abstruscodes-first]{:target="_blank"} ?  Nous avons maintenant la
preuve qu'une entité Doctrine peut commander de la pizza. *CQFD*.


[abstruscodes-previous]:      {{ site.url }}fr/dependency-injection-with-doctrine-part-2.html "Article précédent"
[abstruscodes-first]:         {{ site.url }}fr/dependency-injection-with-doctrine-part-1.html "Premier article de la série"

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