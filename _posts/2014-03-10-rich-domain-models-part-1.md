---
layout: post
title:  Dependency injection with Doctrine - Part 1
---

This post series is about building a rich domain model using [Symfony](http://symfony.com/) and the
[Doctrine ORM](http://www.doctrine-project.org/).  It's not about business analysis or UML.  It a
collection of tools and techniques to help coders think of their framework as an tool, not as a
bunch of bounds and limitations.

*Rich domain model* may have different meanings from one project to another.  Let's define it as 
the opposite of an 
[*anemic domain model*](http://www.martinfowler.com/bliki/AnemicDomainModel.html).  In 
[this article](http://www.martinfowler.com/bliki/AnemicDomainModel.html) (by the way, I strongly
recommend to read it), Martin Fowler uses the following words :

> There are objects, many named after the nouns in the domain space, and these objects are 
> connected with the rich relationships and structure that true domain models have. The catch 
> comes when you look at the behavior, and you realize that there is hardly any behavior on these 
> objects, making them little more than bags of getters and setters.

We've all seen in it.  A domain model is a tool and may not be the best in every situation.
Let's suppose that we need one in a given situation, or at least we want to be able to make one
work, *just in case*.  Using the Symfony framework, that may imply that you want to implement
complex behaviors within your entities.

But how can we inject dependencies in doctrine entities ?

There is no obvious workaround and nearly no documentation of the topic.  It is not possible, 
strictly speaking, but solutions do exists. The good news are that Symfony and Doctrine are better 
frameworks than what their documentations show at first glance.  As 
[Benjamin Eberlei stated in his blog:](http://whitewashing.de/2013/09/04/decoupling_from_symfony_security_and_fosuserbundle.html)

> [...] a framework has to be evaluated by how much it allows you to hide it from your 
> actual application.

Even if the documentation of both [Symfony](http://symfony.com/doc/current/index.html) and 
[Doctrine](https://doctrine-orm.readthedocs.org/en/latest/) obfuscate those features, they
exist.  In facts, they don't.  But the boilerplate code needed to make it work is quite
flyweight and easy to figure out once you got the idea.

Before we dive any deeper in the details, I recommend to read a bit about the 
[SOLID principles](http://en.wikipedia.org/wiki/SOLID).   William
Durand has [a good post](http://williamdurand.fr/2013/07/30/from-stupid-to-solid-code/) on the topic
and [Clean Code](http://www.amazon.ca/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882) is 
a must-have book.

I keep code samples [available on github](https://github.com/abstrus/AbstrusRichModelBundle). 
**Don't expect that code to work out of the box yet.**  Do not *copy-waste* it like an idiot !

## To CRUD or Not To CRUD

I will not try to convince anyone whether a 
[CRUD](http://en.wikipedia.org/wiki/Create,_read,_update_and_delete)-based architecture is *bad* or
not.  We're all tempted to start a project as a CRUD application.  Reason for this are numerous,
namely

- The framework documentation exposes mainly CRUD examples
- The team have experience in CRUD architecture
- What else (but create, read, etc.) would you want that application do ?
- What's wrong with CRUD anyway ?
- We're not starting, the application is already a CRUD one.
- CRUD architecture is conceptually close to 
[transaction scripts](http://martinfowler.com/eaaCatalog/transactionScript.html) which sounds familiar to developers having a *data* background
- etc.

The fact is that CRUD may not be enough.  Plus, what makes a CRUD architecture so fast to implement
is that it often uses strong convention (or configuration, for that matter).  A CRUD application 
well woven with conventions boils down to really simple methods and *heavy* metadata definition.
Ever-changing business logic will make you hate configuration files and metadata definitions.  Code
reuse becomes difficult to implement with configuration, and not all nice design patterns made 
possible by object-oriented programming can make it through. CRUD leads to **opinionated software**, 
which may or may not be what you need.

You may have found this post searching nice buzzwords like *design pattern* and *domain driven design*
in hope for solutions.  If possible, solutions bringing *interoperability* between your existing
CRUD code and the beautiful value-bearing code you are about to write.  This is the topic I'll try
to address in this series.

## Rich domains

So you would really like to have a `$company->hire($aPerson)->asAContractor()` ?  If `$company` must
be retrieved from a persistence layer, you may have a hard time weaving some dependency injection in 
it. Let's use that `$company` Doctrine entity as an example.

{% highlight php startinline %}
// AcmeBundle/Entity/Company.php
class Company {
    protected $name;
    protected $staff;
    
    public function __construct()
    {
        $this->staff = new ArrayCollection();
    }
    
    // Getters, setters, adders, removers, and such
}
{% endhighlight %}

We have a plain data structure with all usual methods that ease the use of Symfony forms, doctrine
components, and so on. This is not really object oriented programming because you'll end up having 
*do-er* classes handling that *know-er* class.  C programmers could call this an hypocrite `struct`. 
Let's start over and make two classes.

{% highlight php startinline %}
// AcmeBundle/Model/Entity/Company.php
class Company {

    protected $value;
    
    protected $humanResourcesService;
    
    public function __construct($value, HumanResourcesService $humanResources)
    {
        $this->value = $value;
        
        $this->humanResourcesService-> $humanResources;
    }
    
    public function hire(Person $aPerson)
    {
        // Initialize passwords, create corporate email, etc.
        $newEmployee = $this->humanResourcesService->createEmployeeFrom($aPerson);
        
        // Send invitations, order pizzas ...
        $this->humanResourcesService->organizeWelcomePartyFor($newEmployee);
        
        $this->value->staff->add($newEmployee);
        
        return $newEmployee;
    }
}
{% endhighlight %}

{% highlight php startinline %}
// AcmeBundle/Model/Entity/CompanyValue.php
class CompanyValue {

    public $name;
    
    public $staff;
    
    public function __construct()
    {
        $this->staff = new ArrayCollection();
    }
}
{% endhighlight %}

I still have a *know-er* class (`CompanyValue`), but now I also have one *do-er* class to rule 
them all.  In facts, `Company` and `CompanyValue` are one (one to rule them all, you got it).  Don't 
tell me it's naughty to expose public properties in this way.  Public properties are  functionally 
equivalent to plain getters and setters.  If you are not happy with it, fill it with getters and 
setters, it'll work. More so, that `CompanyValue` class is intended to be what other languages may 
call a protected, nested  or inner class.  So yes, this is a pure `struct`, not an hypocrite one. 
Some may call it a  *database abstraction layer*, which is quite true.  We may later find a way
to get completely rid of that data structure.

Let's focus on the `Company` class.  This is our domain object.  It's `$value` property could be any
kind of container (plain array, parameter bag, and so on).  The `Company` class does not know about
the persistence layer.  It can be hydrated from a database, configuration, remote web service, 
anything you can think of.  In our case, `$value` will be a `CompanyValue` instance.

The `CompanyValue` implements a data container to be used as a Doctrine entity.  This is the 
class of objects you'll store in a database, be it relational or not.  We'll see later that our 
`Company` could have multiple `$value` property, implemented by multiple `SomethingValue` classes.  
Let's stick with that simple case for now.

We don't want to end up manually hydrating the `Company` with a `CompanyValue` from within our 
controllers. We do want it to be correctly initialized with associated dependencies 
(a `HumanResourcesService`, in that case) and value directly when we retrieve it the way we are 
used to : using an object repository. 

The Symfony documentation taught us to `findBy` our entities, feed them to `Forms` and `persist` 
them with an entity manager.  There lies our first challenge : using our new `Company` with all the 
nice features Symfony offers.

That will be enough for this post : we defined what we aim for and we identified a first issue.  
Next on, we will tackle the Doctrine repository.

