---
layout: post
title:  Injection de dépendance avec Doctrine - Première partie
categories: fr
---

Dans cette série sont présentés des outils pour enrichir le *modèle métier* d'une application
construite avec [Symfony][symfony-home]{:target="_blank"} et l'[ORM Doctrine][doctrine-home]{:target="_blank"}.
Il n'est pas question d'analyse d'affaire ou d'UML.  Nous découvrons les forces de nos *frameworks* 
plutôt que de n'en voir que les limites.

D'un projet à l'autre, *avoir un modèle métier riche* peut avoir un sens variant.  Définissons-le
comme l'opposé d'un *modèle métier anémique* tel que [présenté par Martin Fowler][fowler-anemic]{:target="_blank"}.
D'ailleur, je recommande chaudement cet article.  Il y est d'ailleur écrit :

> There are objects, many named after the nouns in the domain space, and these objects are 
> connected with the rich relationships and structure that true domain models have. The catch 
> comes when you look at the behavior, and you realize that there is hardly any behavior on these 
> objects, making them little more than bags of getters and setters.
>
> > Ils y a des objets, dont plusieurs sont nommés en fonction des termes du métier, et ces objets 
> > sont reliés dans une riche structure de relations que les vrais modèles métiers ont.  L'attrape
> > se découvre quand on regarde le comportement et qu'on réalise qu'il n'y a pas vraiment de 
> > comportements implémentés dans ces objets, les réduisant à rien de plus qu'un tas d'accesseurs 
> > et de mutateurs.
> {: .free-translation}

Des classes de composées presque exclusivement de *getters* et *setters*, nous en avons tous vues.
Nous en avons tous écrites ou générées avec des 
[outils du *framework*][symfony-generateentity-fr]{:target="_blank"}.  Un modèle métier riche reste
simplement un outil, pas nécessairement le meilleur en toutes circonstances.  Pour cet article,
supposons que nous avons besoin d'un tel outil que ce soit pour utiliser directement dans le code de 
*production* ou simplement pour en apprendre assez afin se sentir apte à s'en servir au besoin.

> D'accord, je vais de coder un *modèle métier* riche.  À quoi ça ressemble ?

Votre *modèle-métier* dépendra du ... métier !  Nous utiliserons un exemple simpliste : 
*« Une entreprise embauche un nouvel employé en tant que consultant »*.  Une entreprise
étant stockée en base de donnée, elle est implémentée par une entité doctrine. Éventuellement, nous
devrons faire appel au *service des resources humaines*.  Comment pouvons nous avoir accès au 
conteneur de services au sein même d'une entité doctrine ?

Il n'y a pas de solution facile ni beaucoup de documentation sur le sujet.  C'est en fait impossible
sans remplacer certains éléments du DoctrineBundle.  La bonne nouvelle est que même si la 
documentation est évasive sur le sujet, Symfony et Doctrine fournissent des outils suffisants.

Il y a des exemples de codes plus charnus [sur Github][github-richmodel]{:target="_blank"}. 
**Ne supposez pas que le code fonctionnera directement.**  Ce ne sont que des exemples et des 
brouillons.

## CRUD ou ne pas CRUD

Je ne suis pas de ceux qui croient qu'une application un minimum complexe peut se résumer aux
quatre opérations élémentaires pour la persistance des données.  Une telle application est souvent
issue d'une approche *données d'abord* et se réduit à une base de donnée décorée de formulaires.
Plusieurs choses poussent à orienter le développement d'une nouvelle application avec cette approche:

- La documentation des *frameworks* présente principalement des exemples CRUD-esques
- L'équipe est à l'aise avec des architecture CRUD
- Quoi d'autre que *créer*, *lire*, *mettre à jour* et *supprimer* l'application a-t-elle besoin de faire ?
- Qu'y-a-t-il de mal avec CRUD, au fait ?
- Ce n'est pas une *nouvelle* application et le code existant est déjà CRUD-esque
- Une architecture CRUD est conceptuellement proche des 
[scripts transactionnels][fowler-transaction]{:target="_blank"}, une chose avec laquelle les 
développeurs ayant une bonne expérience en base de données sont à l'aise.
- etc.

Écrire du code autour de ces seules quatre opération n'est pas mal en soi.  De plus, on arrive
à développer de tels programmes très rapidement, surtout si on utilise de fortes conventions 
laissant notre *framework* exécuter toutes sortes de tours de magie nous facilitant la tâche.
L'application comporte finalement surtout des définition de métadonnées sur les classes et les
relations.  Presque toute la logique d'affaire est sérialisée en configuration, laquelle devient
lourde et laisse peu de place à la réutilisation de code et aux patrons de conceptions.  Une
telle approche mène à du logiciel *entêté* (*opinionated software*), ce qui est peut-être ce qui 
vous sied.

Supposons donc que nous ne voulons pas nous restreindre à une base de données ornée de CSS et de
formulaires.  Tentons de trouver des solutions facilement intégrables dans du code existant.

## Domaines métiers riches

Vous voudriez pouvoir faire en sorte qu'une `$compagnie->embauche($unePersonne)->enTantQueContractuel()` ?
Si cette `$compagnie` doit être extraite d'un système de persistance de données,  nous aurons bien
des misères à y injecter des dépendances et implémenter l'*inversion de contrôle* que nous désirons.
Prenons l'entité Doctrine `$compagnie` en exemple.


{% highlight php startinline %}
// AcmeBundle/Entity/Company.php
class Companie {
    protected $nom;
    protected $employes;
    
    public function __construct()
    {
        $this->employes = new ArrayCollection();
    }
    
    // Accesseur, mutateurs et compagnie
}
{% endhighlight %}

Nous avons alors une structure de données quelconque munie de toute les méthodes usuelle qui 
facilitent l'utilisation des formulaire de Symfony et autres composantes.  Ce n'est pas vraiment
une class *orientée objet* puisque nous auront besoin d'objets *qui agissent* pour prendre en charge
cette classe *qui connait*.  Le principe de base de la programmation orientée objet étant que un
objet est le seul responsable de son état (ces données), ceci est tout simplement *du code qui pue*
(tentative de traduction libre de *code smell*.  Un programmeur C appellerait une telle classe une
*struct* d'hypocrite.  Faisons table rase et recommençons du début.

{% highlight php startinline %}
// AcmeBundle/Model/Entity/Compagnie.php
class Compagnie {

    protected $sauvegarde;
    
    protected $ressourcesHumaines;
    
    public function __construct($sauvegarde, ServiceDesRessourcesHumaines $ressourcesHumaines)
    {
        $this->sauvegarde = $sauvegarde;
        
        $this->ressourcesHumaines-> $ressourcesHumaines;
    }
    
    public function embaucher(Personne $unePersonne)
    {
        // Initialisation des accès, courriel professionnel, etc.
        $nouvelEmploye = $this->ressourcesHumaines->creerEmployeAPartirDe($unePersonne);
        
        // Envoi d'invitation, commande de pizza, etc.
        $this->ressourcesHumaines->organiseUneFeteDeBienvenuePour($nouvelEmploye);
        
        $this->sauvegarde->employes->ajouter($nouvelEmploye);
        
        return $nouvelEmploye;
    }
}
{% endhighlight %}

{% highlight php startinline %}
// AcmeBundle/Model/Entity/CompanieSauvegarde.php
class CompanieSauvegarde {

    public $nom;
    
    public $employes;
    
    public function __construct()
    {
        $this->employes = new ArrayCollection();
    }
}
{% endhighlight %}

Nous avons toujours une classe *qui connait* (`CompanieSauvegarde`) et une classe *qui fait*. Cette
dernière nous impose par contre aucune limite.  Nous pouvons considérer qu'elles ne font qu'une.
Ne me dites pas que c'est vilain d'exposer publiquement les propriétés de la classe 
`CompanieSauvegarde`.  Ces propriétés publiques sont fonctionnellement équivalentes à une série de
simples accesseurs et mutateurs.  Si cela ne vous plais pas, ajoutez ces méthodes inutiles. 
Considérons la class `CompagnieSauvegarde` comme ce que d'autres langages nomment des classes 
privées, internes ou [*amies*][wiki-friendclass]{:target="_blank"}.  En C++, on dit que les classes
amies peuvent améliorer l'*encapsulation* dans le cas ou deux partie d'un même classe n'ont pas
nécessairement le même nombre d'instance ou la même durée de vie, or c'est exactement ce que nous
voulons accomplir.

Concentrons-nous sur la classe `Compagnie`.  C'est notre objet métier.  La propriété `$sauvegarde`
pourrait être n'importe quel conteneur (un simple tableau, par exemple).  La classe `Compagnie` est
totalement indépendante de la couche d'accès aux données persistantes.  Sa sauvegarde peut provenir
de n'importe où, que ce soit d'une base de donnée, de la configuration ou d'un service distant.
Pour notre exemple, `$sauvegarde` sera une instance de `CompagnieSauvegarde`.  `CompagnieSauvegarde`
est un conteneur de donnée qui sera pris en charge par Doctrine.  Nous allons aborder plus tard
comment nous pourrions avoir plusieurs propriétés comme `$sauvegarde` ou aucune.

Nous ne voulons pas avoir à systématiquement injecter manuellement la sauvegarde dans l'objet métier
au sein même du code d'un contrôleur.  Nous voulons qu'il soit automatiquement initialisé, incluant
les dépendances associées (telles qu'un service des ressources humaines, par exemple).  De plus,
nous ne voulons pas changer la manière de récupérer les objets métiers.  Pour rester cohérent avec
le reste du code de l'application, nous allons utiliser les dépôts de Doctrine (*entity repositories*).

La documentation Symfony nous a habitué à utiliser la méthode `findBy` et ses semblables pour 
récupérer nos objets métiers, les donner à modifier pas des classes `Form` et ensuite de les
sauvegarder avec un gestionnaire d'entités (*entity manager*).  Ceci est notre premier défi :
utiliser notre nouvel objet métier `Compagnie` avec toutes les fonctionnalités usuelles de Symfony.

Nous avons identifié un but et un premier problème.  Prochainement, nous nous attaquerons aux dépôts
de Doctrine (*entity repositories*). 

[github-richmodel]: https://github.com/abstrus/AbstrusRichModelBundle "AbstrusRichModelBundle on Github"

[doctrine-home]: http://www.doctrine-project.org/ "Doctrine Project Homepage"
[doctrine-doc]: https://doctrine-orm.readthedocs.org/en/latest/ "Doctrine Documentation"

[symfony-home]: http://symfony.com/ "Symfony Project Homepage"
[symfony-doc]: http://symfony.com/doc/current/index.html "Symfony Documentation"
[symfony-generateentity-fr]: http://symfony.com/fr/doc/current/bundles/SensioGeneratorBundle/commands/generate_doctrine_entity.html

[wiki-solid]: http://en.wikipedia.org/wiki/SOLID "SOLID Principles - Wikipedia"
[wiki-crud]: http://en.wikipedia.org/wiki/Create,_read,_update_and_delete "CRUD - Wikipedia"
[wiki-friendclass]: http://en.wikipedia.org/wiki/Friend_class "Friend class - Wikipedia"

[willdurand-solid]: http://williamdurand.fr/2013/07/30/from-stupid-to-solid-code/ "William Durand about SOLID principles"

[amazon-cleancode]: http://www.amazon.ca/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882 "Clean Code - Amazon"

[fowler-anemic]: http://www.martinfowler.com/bliki/AnemicDomainModel.html "Martin Fowler about Anemic Domain Model"
[fowler-transaction]: http://martinfowler.com/eaaCatalog/transactionScript.html "Martin Fowler about Transaction Scripts"

[eberlei-framework]: http://whitewashing.de/2013/09/04/decoupling_from_symfony_security_and_fosuserbundle.html "Decoupling from Symfony Security and FOSUserBundle - Whitewashing Blog"