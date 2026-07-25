+++
date = '2026-07-25T15:56:55-03:00'
draft = true
title = 'Dart Getters and Setters'
+++

Hello. I want to talk a litte about the design of Dart's getters and setters. More specifically, the choices the Dart team made when designing getters and setters into the language, and how they help in encapsulating your code without creating uneeded verbosity.

Also, just to be clear, I will be talking about getters and setters together, but most of my examples will be with getters, since they are the most common use case, and in my opinion, when using Dart you should rpefer immutable classes, and that exclude the need for setters.

## What are Getters and Setters?

From the basics, with no programming language specific implementations in mind, getters and setters are a concept in programming on how you should encapsulate the state of your classes/structs.

I want to be clear, because if you do not understand the reason why getters and setters exist, you will probably not understand the reason Dart designed them that way.

As all good OOP examples, let's design a simple Animal class, that holds an `age` state.

(Note: This is just some pseudo-code , just bear with me)

```
class Animal {
  public int age;
}
```

Right now, age is public, so anyone can access it directly.

```
function printAnimalAge(Animal animal) {
  print(animal.age);
}

final animal = Animal(age: 5);
printAnimalAge(animal); // Output: 5

animal.age = 3;
printAnimalAge(animal); // Output: 3
}
```

And this works fine, we can get and set the `age` field of our class, and it compiles fine for the caller and the callee. But, imagine that the project constraints changed, and now we need to store the animal's date of birth instead ot it's age. This can be easily changed for our classÇ

```
class Animal {
  public Date birthDate;
}
```

But how about the consumer of our class?

```
function printAnimalAge(Animal animal) {
  print(animal.age); // Compilation error: `age` does not exist.
}
```

Since the constraints of ourt class changed, **and we didn't encapsulate those constraints**, the consumers of our class suffer, because they were operating on the assumption that our constraints would never change.

That's where getters and setters come in.

So now, we go back in time, knowing this problem. And we need a way for the consumer of our class to know the age of the animal. Remember, that the consumer does not care how the Animal class calculates the age, they just want to know the age.

First, we can make the `age` field private, so that the consumer of our class cannot access it directly.

```
class Animal {
  private int age;
}
```

Good, now the consumer of our class cannot access the `age` field directly. But we still need to allow the consumer to actually get the age of our animal. So, we create a getter for the `age` field.

```
class Animal {
  private int age;

  int getAge() => this.age;
}
```

Now, the consumer of our class can get the age of our animal using the `age` getter, without having to access the `age` field directly.

```
function printAnimalAge(Animal animal) {
  print(animal.getAge());
}
```

Now, when our constraints change, and we need to update our class to use the date of birth instead:

```
class Animal {
  private Date birthDate;

  int getAge() {
    // calculate age based on birth date
  };
}
```

We don't need to update the consumer of our class when we change the implementation, because the `getAge` method is still available.

```
function printAnimalAge(Animal animal) {
  print(animal.getAge());
}
```

This is why getters and setters are so useful. They help us encapsulate implementation details from the consumer of our class. So, when we need to change the implementation, the consumer does not need to change.

## Verbosity

Unfortunately, the story does not end here. Since we understood why getters and setters are useful in helping us encapsulate our implementation, we start implementing them for every field we have, since, if they were not there, the consumer of our class would have to access the fields directly, and we know why we wouldn't want that.

So, for every field we have, there is a getter and a setter.

```
class Animal {
  private int age;

  int getAge() => this.age;
  void setAge(int age) => this.age = age;
}
```

But, what if our class has a lot of fields? Well, we still want them behind getters and setters, even if there is no need to add any logic to them.

```
class User {
  private String name;
  private int age;
  private String email;
  private String address;
  private List<String> phoneNumbers;
  private Map<String, String> socialMedia;
  private Uri profilePictureUrl;

  // getters and setters for each field
}
```

As you can see, the class grows, and with it, the number of getters and setters. Getters and setters that, for itself, do not contain any custom logic, just redirecting to the field.

And if you worked in codebases like these, you know: the class grows. And now, the class that you use for holding data, has a ton of getters and setters that just polute the class, adding maintainability and verbosity for a class that should be simple and straightforward.

## Dart's Solution

To be clear, Dart is not the first language that tackles this problem. A lot of languages use some way to automatically generate getters and setters for you. I just want to show how Dart handles this, and why I think it's a pretty good design.

Let's go back to our Animal example, with it's 