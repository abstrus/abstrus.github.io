---
layout: post
title:  Curing Domain Model Anemia - Part 3
---



## Half Repository, Half Factory

We are not over yet.  Our custom repository need to return `Company` objects, not `CompanyValue` objects.
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
there is a need to paginate the results.  

This part may be a bit tricky.  It really depends on how the entity repository is intended to be used.
As it provides way more flexibility and power without adding much complexity, I would recommend to
**never use** the `find*` magic methods and use the QueryBuilder and Query objects instead.  This 
[post](http://whitewashing.de/2013/03/04/doctrine_repositories.html) from Benjamin Eberlei provides a 
really nice start for using a variant of the 
[specification pattern](http://en.wikipedia.org/wiki/Specification_pattern) with Doctrine.  More so,
the data hydrated using the magic methods do not use the same workflow as with DQL.  I am far from
being a Doctrine-guru, but I don't like when there are many ways of doing the same thing.

Back to our business, we need to hydrate a Company object with services and a CompanyValue.  To 
hydrate things, I need a `Hydrator`.  

## Right Hydration for a Thirsty Business Model

If you look down in Doctrine's internals, you should find
some [hydrators](http://www.doctrine-project.org/api/orm/2.4/namespace-Doctrine.ORM.Internal.Hydration.html).
