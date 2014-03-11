---
layout: post
title:  Curing Domain Model Anemia - Part 1
---

I have been trying to figure out how to cure an 
[anemic domain model](http://www.martinfowler.com/bliki/AnemicDomainModel.html) lately. Using 
[Symfony](http://symfony.com/) and the [Doctrine ORM](http://www.doctrine-project.org/), I tried to 
have persistent entities to be something more than plain data structures with *getters* and 
*setters*.  The fact is that's a tremendous task, especially when you work with an existing 
application (also known as *legacy code*).

There are some really interesting articles out there about
[*Domain Driven Design*](http://martinfowler.com/tags/domain%20driven%20design.html).  I will not
tell you I did DDD lately, that would be a lie.  DDD is hard to implement when the developer team, 
the boss or the client is not used to it.  I am not used to it either, but I do want to write
good, valuable, maintainable and filled with value code.  This post and some more later on are
intended to provide pragmatic tips and tricks on feeding some iron in an anemic business model. 
I will not directly implement any of the \*DD 
([TDD](http://fr.wikipedia.org/wiki/Test_Driven_Development), 
[BDD](http://fr.wikipedia.org/wiki/Behavior_Driven_Development), 
[DDD](http://en.wikipedia.org/wiki/Domain-driven_design), etc.).  Consider these posts as ideas
to improve your code base and inspirations you can adapt to you own situation.

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
Some may call it a  *database abstraction layer*, which is quite true.

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

