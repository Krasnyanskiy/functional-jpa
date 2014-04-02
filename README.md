functional-jpa
==============

Functional style helpers for jpa and guava (tests fine with 16.0). Has an optional dependency 
on rx-java if you wish to use Observables (which are very cool!).

Status: *pre-alpha*

Features
-------------------
* method chaining for EntityManager, EntityManagerFactory 
* improved Query builder
* lazy iteration and paging of result sets (great for large result sets)
* can return result set as Guava FluentIterable 
* can return result set as *rx-java* Observable
* RichEntityManager.run handles try catch final noise for closing resources and logging (slf4j)
* Funcito helper methods

Query iterators
------------------

*Rich* versions of `EntityManager.createQuery` method have fluent style and enable lazy iteration 
of the result set of a query (which uses `setFirst` and `setMaxResults` for paging under the covers).

To get a rich version of an `EntityManagerFactory`:

```java
    EntityManagers.enrich(normalEmf);
 ``` 
    
Now to an example:

Given this jpa class (note the use of [Funcito](https://code.google.com/p/funcito/) to create the guava function for id):

```java
@Entity
public class Document {
    @Id
	private String id;

	public Document(String id) {
		this.id = id;
	}

	public String getId() {
		return id;
	}

	@Column
	public String status;

	public static Function<Document, String> toId = functionFor(callsTo(
			Document.class).getId());
}
```
You can do stuff like this:

```java
import com.github.davidmoten.fjpa.EntityManagers;

RichEntityManagerFactory emf = EntityManagers.emf("test");
RichEntityManager em = emf.createEntityManager();

// get a list of all ids in documents
List<String> list =
    em
       .createQuery("from Document order by id",String.class) 
	   .pageSize(2000)     //default page size is 100
	   .fluent()           //as FluentIterable
	   .transform(toId)  //get id (lazily)
	   .toList();          //force evaluation to list
```

or using Java 8 lambdas:
```java
// get a list of all ids in documents
List<String> list =
    em
       .createQuery("from Document order by id",String.class) 
	   .pageSize(2000)     //default page size is 100
	   .fluent()           //as FluentIterable
	   .transform(d -> d.id)  //get id (lazily)
	   .toList();          //force evaluation to list
```

Eliminating try-catch-final noise
---------------------------------------
You can also get the `RichEntityManagerFactory` to perform all of the usual try-catch-final 
closing of resources and logging of errors using the `RichEntityManagerFactory` run method:

```java
RichEntityManagerFactory emf = EntityManagers.emf("test");
List<String> list = 
		emf.run(new Task<List<String>>() {
			@Override
			public List<String> run(RichEntityManager em) {
				return em
						.persist(new Document("a"))
						.persist(new Document("b"))
						.persist(new Document("c"))
						.createQuery("from Document order by id",
								Document.class).fluent()
						.transform(toId).toList();
			}
		});
assertEquals(newArrayList("a", "b", "c"), list);
emf.close();
```

or using method chaining even further for the same result:

```java
emf("test") 
  .run(new Task<List<String>>() {
	@Override
	public List<String> run(RichEntityManager em) {
		return em
				.persist(new Document("a"))
				.persist(new Document("b"))
				.persist(new Document("c"))
				.createQuery("from Document order by id",
						Document.class)
				.fluent()
				.transform(toId)
				.toList();
	}
}).process(new Processor<List<String>>() {
	@Override
	public void process(List<String> list) {
		assertEquals(newArrayList("a", "b", "c"), list);
	}
}).emf().close();
```  

or the same again but using Java 8 lambdas for less noise:

```java
emf("test")
   .run(em -> em.persist(new Document("a"))
				.persist(new Document("b"))
				.persist(new Document("c"))
				.createQuery("from Document order by id",
						Document.class)
				.fluent()
				.transform(toId)
				.toList())
   .process(list ->
		assertEquals(newArrayList("a", "b", "c"), list))
	.emf().close();
```

Funcito helper methods
--------------------------
[Funcito](https://code.google.com/p/funcito/) is a great tool in the absence of Java 8. The methods `FuncitoGuava.functionFor` combined with `FuncitoGuava.callsTo` allow 
wonderfully concise creation of Guava Functions.

For example:
```
Function<Document,String> toId = functionFor(callsTo(Document.class).getId());
```

I love it but its a bit verbose so I added a simple `FuncitoHelper` class to functional-jpa that allows for an abbreviated version:
```
Function<Document,String> toId = f(c(Document.class).getId();
```

Here's an example using *functional-jpa* that is very close in conciseness to using Java 8 lambdas:
```
import com.github.davidmoten.fjpa.EntityManagers;

RichEntityManagerFactory emf = EntityManagers.emf("test");
RichEntityManager em = emf.createEntityManager();

// get a list of all ids in documents
List<String> list =
    em
       .createQuery("from Document order by id",String.class) 
	   .pageSize(2000)     //default page size is 100
	   .fluent()           //as FluentIterable
	   .transform(f(c(Document.class).getId())  //get id (lazily)
	   .toList();          //force evaluation to list
```
