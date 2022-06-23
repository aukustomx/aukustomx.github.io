# Anemic Domain Models

### Modeling beyond getters and setters

.

## Everything we do is about our clients... and the domain
It's always about the customer. Everything we did before, everything we're doing now, and everything we are going to
do in the future, in the software development industry, is about the customer. We and our companies want the customer
to be satisfied with our work and with the goods and services we provide.

Every decision we make must be made with the customer in mind. At every phase of the SDLC (analysis, design,
coding, tests, etc.), we must keep the customer in mind. What kind of experience do we want to offer them?

But, as the title of this section says, in our industry it is also about the domain. What is the domain? Let's see.

## Domain
The domain is the main topic or subject that you are going to model and solve. If your company is a bank, some example of 
domains are:
- **Notifications** you send to the clients
- Types of **accounts** you offer.
- Investment packages
- Promotions
- Etcetera

## Let's make everything domain-driven
If the domain is so important, we have to make all decisions following the domain:
- Abstractions
- Entities
- Main and alternatives flows
- Participants
- Tech stack
- Patterns
- Infrastructure
- Etcetera

Why? Because we are supposed to model the real world, a real domain.

### Subdomains
It's posible that within a domain we can find subdomains. For example, if we talk about bank cards, we can see debit
and credit card. I'm not an expert in banking business, but let's say the business rules of both types of cards are too
complex to be managed with a single domain. So, we can say that we have the card domain and also the debit and credit 
card subdomains. Or in an e-commerce company, we can have Orders, Payment, Shipping, etc. Something like that.

Once we identify those subdomains, we can treat each one as separated domain in itself and apply the thoughts we've
talked about so far.

## A little about DDD
This article is not about DDD. Ok, just a little bit. So, I want to say a few words about it. 
Domain Driven Design or DDD, is an approach or a way of dealing with the design of complex software solutions.
It is important to note the word *complex*. This design must take into account the domain, it must be guided by the
issue we are trying to solve.

It seems quite obvious, but it is not always the case.

DDD is a complex subject in itself. It has concepts and techniques in charge of the business and technology areas.
Things like:

- Ubiquitous language: everyone involved (business and technology teams) needs to speak the same language, use the same terminology
- Bounded contexts: Identify the edges of a domain or subdomain
- Models: entities, value objects, aggregations, etc.
- And many others concepts

## DDD models are closely related to OOP
In OOP world the object is the king, and in DDD models are a crucial part. These two concepts are closely related each other.
I could say that they are more or less the same. Using both we want to create an abstraction of the real world.

But OOP is not just about object data or object attributes. OOP is concerned with the behavior of objects. Think of a car.
If we only design by looking at the attributes of the car, our design will never allows us to use the car, i.e., ask it to drive,
or to stop or to notify us its mechanical status. We will only be able to ask it what its color is, what its brand is, etc.

With this in mind, we could say (the industry says), OOP is about state and behavior.

## We already create models in some ways
With a little of differences, we use something like the following java package structures to put our models in. 
```java
/project/app/web/model
/project/app/domain/entity
```
We call them requests, responses, entities, value objects, etc. But in any case, what we are trying to do is to represent real
world objects.

## Are the following models familiar to us?
Suppose we want to design a solution to manage the account that a bank customer may have. I will show you our first
anemic domain models. I will use the class name prefix *Anemic* to distinguish these models:

AnemicCard.java
```java
public class AnemicCard {

    private String cardNumber;
    private String cvv;
    private Date expirationDate;
    
    //getters and setter...
}
```
AnemicAccount.java
```java
public class AnemicAccount {

    private String accountNumber;
    private String accountType;
    private String creationDate;
    private List<AnemicCard> cards;

    //getters and setters...
}
```

AnemicClient.java
```java
public class AnemicClient {

    private String name;
    private String clientId;
    private String email;
    private List<AnemicAccount> accounts;
    
    //getters and setters...
}
```

## Why are these models considered anemic?
We can find many problems with these models: 
- Where is the behavior?
- Vulnerable state
- Primitive obsession
- Duplicated code
- Imperative object creation steps
- Imperative code in general

Let's break down each of the problems one by one. The party is just getting started.

## Primitive obssesion
If we are using an object-oriented language, why are we so obsessed with using primitives and not rich, complex objects? 

In our code, let's review the class `AnemicAccount` and its `accountType` field. Why do we use an `String` to represent
the type? Maybe the account types in the bank are well-defined, i.e., it could be a fixed catalog. If it is the problem,
someone might say: let's use a constant class to define all possible values.

It could look like this:
```java
public class Constants {
    
    public static final String accountType1 = "type1";
    public static final String accountType2 = "type2";
    public static final String accountType3 = "type3";
}
```
This solution is worse than the previous one. It spread the state outside of the class that is interested with those types,
and, from a Java class design perspective, this constant class is not taking care of things utility classes should.

## Don't be afraid, let's use rich objects
If account types were a fixed catalog, we can use a Java `enum`:

With this, we can add attributes and behavior to the `AccountType` class, like description, tax regime, etc.

This design gives us the posibility to get all the `AccountType`s designed for People and not for companies?
```java
public enum AccountType {

    CHECK("Bank checks emissions", "PFAE", "Companies"),
    SAVING("Client saving account", "PF", "People"),
    CHILDREN_SAVING("For children money saving", "No regime", "People");

    private String description;

    //This field could be a complex object
    // or an enum by itself
    private String taxRegime;

    //This field could be a complex object
    // or an enum by itself
    private String segment;

    AccountType(String description, String taxRegime,
                String segment) {
        this.description = description;
        this.taxRegime = taxRegime;
        this.segment = segment;
    }
    
    //only getters...
    
    //a convenient method to get all types of
    // account intended for a specific segment
    public static List<AccountType> bySegment(String segment) {
        return Arrays.stream(values())
                .filter(accountType -> accountType.getSegment().equals(segment))
                .collect(Collectors.toList());
    }
}
```

Now, let's think about the need to apply validations to the `accountNumber` at the time of `Account` object creation. 
To accomplish this in a correct OOP manner, we could transform `accountNumber` from a `String` to a rich model/complex object, something
like this:

```java
//This is a value object. It hasn't an identity. It just holds a value. 
public class AccountNumber {

    private final String number;

    private AccountNumber(String number) {
        this.number = number;
    }

    //Using a factory method. It allows us many advantages such as validations, reuse of
    // heavy object creation, etc. It saves us to write the new keyword
    public static AccountNumber of(String number) {

        //Here we can put validations
        //The number lengh, the content, not numbers, not null, etc.
        valid(number);
        return new AccountNumber(number);
    }

    private static void valid(String number) {
        //Let's start with the name. What validation we need to perform: not null, not empty,
        // at least two word separated by a space, not numbers, etc.
        if (isNull(number)) {
            //log something
            throw new IllegalArgumentException("The account number can't be null");
        }

        if (number.isBlank()) {
            //log something
            throw new IllegalArgumentException("The account number can't be empty");
        }

        if (!number.matches("[0-9]{11}")) {
            //log something
            throw new IllegalArgumentException("The account number must be eleven numbers");
        }

        //Etc, etc, etc. Validate here any other rule on name and other params
    }

    //Only accesors. When an accountNumber born, it's supposed to never change.
    public String getNumber() {
        return number;
    }
}
```

Let's see what the creation validation of an AccountNumber looks like usign both anemic and rich domain models:

```java
import com.aukustomx.richmodels.AccountNumber;

public class AccountService {

    //Anemic way. All the logic of the validations inside the domain service
    public Object anemicCreateAccount(String number) {
        //Validate account number param before we use ot to create an AnemicAccount
        var number = "12343";
        if (isNull(number)) {
            //log something
            throw new IllegalArgumentException("The account number can't be null");
        }

        if (number.isBlank()) {
            //log something
            throw new IllegalArgumentException("The account number can't be empty");
        }

        if (!number.matches("[0-9]{11}")) {
            //log something
            throw new IllegalArgumentException("The account number must be eleven numbers");
        }

        //Once we have validated everything is needed to, we proceed to create 
        // our anemic object using our recently validated number.
        var anemicAccount = new AnemicAccount();
        anemicAccount.setAccountNumber(number);
        var accountNumber = AccountNumber.of("ABCD");
        //...
    }

    //Rich domain models way. All validations are within the object itself.
    public Object richCreateAccount(String number) {
        var accountNumber = AccountNumber.of(number);
        //...
    }
}
```
Can you see the differences?

## Don't use empty constructors, when you can
In the real world, objects have some characteristics that are impossible to be changed (at least not legally). Let's say that 
we are modelling a Person. A Person have a date of birth, a name, a nationality, an identity document, 
an address where he/she lives, a job, and so on.

All of these attributes of the person, there are some that will never change no matter what happens. In our example, those 
characteristics could be birthday, name and nationality.

So, why do we insist on modeling that Person with a class that allows changing those values whenever the user of that
class wants?

Let's see this in code.
```java
public class AnemicPerson {

    private String name;
    private Date birthday;
    private String nationality;
    private String address;

    //getters and setters
}

public Object createPerson(/*any params*/) {
    
    var anemicPerson = new AnemicPerson(); 
    anemicPerson.setName("Juan");
    anemicPerson.setBirthday(new Date());
    anemicPerson.setNationality("Mexico");
    anemicPerson.setAddress("Here");
    
    //We have validation problems here again
    //Valid name, a person cannot be born in the future, it is assumed that he/she was born in some country on earth
    // not it Helium, Barsoom, you got it?
    
    //This is a valid sentence.
    anemicPerson.setNationality("Helium, Barsoom");
}
```

So we need to protect those attributes, that is the *internals*, the *invariants*, the state of the object.
If we use empty constructors, the only way to create new Person objects is to have setters in every field of our class.

## Stop doing select the class, then right clic, then generate, then getter and setters
No words

## Convenient methods: equals, filtered lists, presentation/logging messages, as many as we need
What if we want to compare two objects according to some of their attributes, or if we want to get a filtered list of all 
countries a person has lived in before, or the last 3 digits of each card number of a bank customer.

To perform these tasks, we can take advantage of the power offered by rich models.

Suppose that we want to log a message representation of our Person object. Let's say that we want to print the date when
our Person was born. But in this case, we are only interested in the day and month.  We have two options: use the anemic
one and the rich one.

Anemic way:
```java
    
    var message = anemicPerson.getName() + " has born on " + anemicPerson.getBirthday().getMonth() + " " +
            anemicPerson.getBirthday().getDay();

    log.info(nessage);

```

Rich way:  
```java
    log.info(person.birthdayLogInfo());
```

## Conclusions
Using rich domain models adds meaninful semantics to our objects, improves abstraction and encapsulation, hides
implementation details from the clients of our classes, reduces duplicated code, protects the invariants of our
objects and gives us many other advantages.

Whth rich domain models our models go from being just data structures to being a true representation of the real world
object, an object with state and behavior.

So, let's stop creating poor, anemic models and start choosing rich models.


Happy coding!

## References
* https://www.martinfowler.com/bliki/AnemicDomainModel.html
* https://thedomaindrivendesign.io/anemic-model/
* https://thedomaindrivendesign.io/anemic-model-x-rich-model/
