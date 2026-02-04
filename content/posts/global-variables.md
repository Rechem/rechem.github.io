---
title: "Global variables are killing your code quality"
date: 2026-02-04T12:00:00+00:00
draft: false
description: "Global variables seem convenient until they're not. I this post, I talk about the issues that arise when using global variables and how to solve them using dependency injection with practical examples."
tags: ["design patterns", "testing", "code smell"]
---


## Introduction

Whether you are a student working on your assignment, or an office clerk writing code for a living, you are tasked with building your next backend application.
Your fingers start rushing over the keyboard, your code base starts growing, and your brain reaches flow state. You start writing your services, define schemas, perform validation and wiring, and then it's time to add a database client. Where do you instantiate it, and how do you use it?

If you were like me, I used to do something like this:
- Create a file for the database client e.g. Prisma: `src/util/prisma_client.ts`
- Instantiate and export the client instance: `export const prismaClientSingleton = new PrismaClient();` 
- Reuse it everywhere

You can actually find this exact code of mine in my [fdaBackend](https://github.com/Rechem/fdaBackend/tree/main) project repository. This was part of an assignment for a food delivery app.

> Credits to me for calling it *Singleton* and using it only in the repository layer right? 🤣

# The problem

While this looks okay on the surface and certainly works, global variable pattern is a code smell and has several issues.

## 1. Hidden dependency

When you're writing code, it must only use explicit dependencies, namely, its arguments and class attributes. This way it stays clean and maintainable, and whenever it's someone else's turn to maintain it or test it, they won't run into unpleasant surprises, like a hidden dependency on a global variable burried deep within the implementation. There are very few exceptions to this rule which include loggers and validators.

## 2. Testability

Global variables break testing entirely. When you write tests for your application (I know you don't), each test must be isolated and independent from the others. Whenever our subject function uses external clients, we create mocks to simulate their behavior during testing - There is also another approach using Test Containers but that will be a topic for another blog.

Using mocks allows our tests to run even without an actual client, whereas when you import the client as a global variable, you have essentially turned your unit tests into integration tests. Let me break it down:

Let's say you have this function
```ts
import { prismaClientSingleton } from '../util/prisma_client';

export default class UserRepository{

	public static getUserById(id: string): Promise<User | null> {
		return await prismaClientSingleton.user.findUnique({
			where: { id }
		});
	}
}
```
When you try to test this function, you can't easily mock the Prisma client because it's imported directly. You have to either use the real database connection (making it an integration test) or resort to hacky solutions like module mocking, which adds a lot of complexity. Your test becomes dependent on the global state, and if multiple tests run concurrently, they might interfere with each other.
Also notice how I use static methods to avoid instantiating objects, another code smell.
## 3. Increased coupling
Global variables create tight coupling between your code and specific implementations. Your business logic becomes directly tied to a particular database client, that's used everywhere in your code. If you want to swap the client and use another ORM, good luck refactoring your entire code base. Every file that imports that global variable is now coupled to that specific implementation, which will cost you hours and headaches to refactor.
# The solution
The solution is very simple, don't use global variables. It's easier than you might think. whenever you need an external dependency like a database client, pass it as a parameter or inject it through a constructor.

## Dependency Injection
The principle is simple, if you need an external dependency, inject it. Do not summon global instances inside your code. if you need a dependency, find a way to inject it.
Here's how we can refactor the previous code:
```ts
import { PrismaClient } from '@prisma/client';

export default class UserRepository {
  constructor(private prisma: PrismaClient) {}

  public getUserById(id: string): Promise<User | null> {
    return await this.prisma.user.findUnique({
      where: { id }
    });
  }
}
```
Here, we inject the dependency through the class constructor. Anyone reading this code immediately sees that `UserRepository` needs a `PrismaClient` to work. Testing also becomes trivial:
```ts
import { UserRepository } from './UserRepository';

describe('UserRepository', () => {
  it('should get user by id', async () => {
    const mockPrisma = {
      user: {
        findUnique: jest.fn().mockResolvedValue({ id: '1', name: 'John' })
      }
    } as any;

    const repo = new UserRepository(mockPrisma);
    const user = await repo.getUserById('1');

    expect(user).toEqual({ id: '1', name: 'John' });
    expect(mockPrisma.user.findUnique).toHaveBeenCalledWith({
      where: { id: '1' }
    });
  });
});
```
Look at this beautiful, clean, testable code. At first glance it may look like we added a lot more code just to make it testable, and if you're the only person working on a one-time project for an assignment, I wouldn't expect you to even think of testing your code. But if you work in a collaborative environment and your code will be deployed in production, the last thing you want is your users testing your code. Moreover this example is oversimplified, in reality your functions will do a lot more work and writing tests for them becomes crucial.

Thanks to dependency injection, we were able to solve all 3 of our problems:
- There are no hidden dependencies, each function uses explicit dependencies injected through the constructor or passed down as arguments.
- The code is now testable.
- And is loosely coupled. If we ever need to swap Prisma for another ORM, there is only one place to edit the code.
## What about the actual client instantiation?

In practice, you still need to instantiate your database client and use it. The key is to instantiate it at your application's entrypoint and pass it down through your dependency chain:
```ts
import { PrismaClient } from '@prisma/client';
import { UserRepository } from './repositories/UserRepository';
import { UserService } from './services/UserService';
import { UserController } from './controllers/UserController';

const prisma = new PrismaClient();
const userRepository = new UserRepository(prisma);
const userService = new UserService(userRepository);
const userController = new UserController(userService);

// Set up your routes with the controller
app.get('/users/:id', (req, res) => userController.getUser(req, res));
```
Yes, this requires a bit more wiring and object initialization at the start, but the benefits outweigh the initial setup cost. Your code becomes testable, reusable and maintainable.

For larger applications, you can use **dependency injection containers** such as [tsyringe](https://github.com/microsoft/tsyringe). These containers do the wiring and injection for you so you'll have far less boilerplate code. Some frameworks, like **NestJS**, go even further by providing **built-in automated wiring** out of the box.

# Conclusion

Global variables might seem convenient in the moment, but they're technical debt waiting to bite you. By using dependency injection and keeping your dependencies explicit, you write code that's easy to understand, test, and maintain. Your future self (and your colleagues) will thank you.

Now go and refactor that code base! 😉
