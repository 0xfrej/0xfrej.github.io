---
title: Handling optional fields ergonomically
description: Optional fields can be cumbersome and quite non-ergonomic to handle, even more so when you have to handle both nullability and their presence. In this small series I will try to explore an idea borrowed from Rust's Option<T> type and implement it using Symfony serializer.
author: freja
date: 2024-10-30 22:51:00 +100
categories: [PHP, Programming, Serialization, Experiments]
tags: [php, programming, serializeration]
---
Whether you're implementing DTO for REST `PATCH` action or handling complex data feed structure which utilizes partial XML for sending incremental updates,
making sure the code is maintainable and ergonomic to write can be taxing. I will mainly focus on PHP, but essentially this can be applied
in any language. 

In this article I will try to illustrate different approaches and my frustrations with them to help you understand how and why have I arrived
at the solution that I like the most so far. I know this article does not contain all possible solutions and it can't possibly without being 
ridiculously long. But I think the ones chosen will serve my purpose well.

I think it is appropriate to say from the start that I do not like to rely on magic features in PHP. They are notoriously hard to maintain, often 
falling out with the reality, rendering static analysis tools useless (if I do, I want to keep it as friendly to static analysis as possible).
As such, you have to rely on possible features of the somewhat existing type system php has. My solution relies on generic annotations, which still isn't 
perfect, but is the best I can have. It is supported pretty well in static analysis tools and the annotations are local on the class members instead 
of like in one of those mega docblocks annotating every memebr on a class in one place (speaking from experience, one old project in our company had classes with docblocks
annotating methods of which about half the function signatures changed and calling them like the annotation suggested would result in a failure).

## Using associative arrays or maps
More often I see optional fields (not to be confused with nullable fields) handled either using raw associative array,
bunch of `if` statements that test whether the field is present in the array sprinkled around and if your `PATCH` handler can work with many fields, it can
grow rapidly and be very unreadable in no time. Not to mention the countless mistakes one can do.

You can also use this approach to write DTOs, but I will get to that later, as I also have some minor issues with it.

```php
#[Route('/api/user/{id}', name: 'api_user_update_by_id', methods: ['PATCH'])]
public updateUserById(int $id, Request $request): JsonResponse
{
    $user = $this->userRepository->getById($id);
    
    if (!$user) {
        return new JsonResponse(['error' => 'User not found'], 404);
    }
    
    $data = $request->toArray();

    if (array_key_exists('email', $data)) {
        if (empty($email)) {
            // handle empty email as error
        }
        $user->setEmail($data['email']);
    }

    if (array_key_exists('firstName', $data)) {
        if (empty($firstName)) {
            // handle empty firstName as error
        }
        $user->setFirstName($data['firstName']);
    }

    if (array_key_exists('lastName', $data)) {
        if (empty($lastName)) {
            // handle empty lastName as error
        }
        $user->setLastName($data['lastName']);
    }

    $this->userRepository->save($user);

    return new JsonResponse(['status' => 'User updated successfully'], 200);
}
```

## Using nullability
Other times I see DTOs that utilize nullability for handling presence of a field. This is fine, until you hit a field that needs to be both nullable and you need
to know if it is present. Which can result in undefined behavior. Say you have an entity `User` with a field `dateOfBirth` that is nullable, because you don't require
your users to specify it. Now take that field in a `PATCH` request:


| **State of `dateOfBirth`** | **Request Action**                  | **Outcome**                                                                                           |
| :-------------------------- | :---------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **Not Set**                | `dateOfBirth` not included         | No change; `dateOfBirth` remains unset.                                                               |
|                             | `dateOfBirth` included with date   | Sets `dateOfBirth` on the `User` with the provided date.                                              |
| **Already Set**            | `dateOfBirth` not included         | **Option 1**: Keep the existing `dateOfBirth`. (prohibits user from ever removing the date from their account) <br> **Option 2**: Clear (set to `null`).              |
|                             | `dateOfBirth` included with date   | Updates `dateOfBirth` on the `User` with the new date from the request.                               |

You can think that this is just a minor issue, frontend can handle it by always sending the value from the entity even if it was not changed, thus allowing users
to also unset the `dateOfBirth` field. However, this not only breaks the conventions tied to the `PATCH` method, it will come with even more issues when the caller is 
not in your hands. 

### Let's think of a data feed for a second
This feed has many, many fields. One of them is some very important value that is nullable. You will receive
updates from this feed in partial form, meaning you now have to handle presence as well. 

Suppose that one day entity `A` had an update in their database and it changed
from something to `null`. The update will also have the field set to `null`, but since our code does not know difference between field being present, and we most likely don't 
want to reset it each time you get an update, meaning that now this field didn't get set to `null` in our system after the update and the entity `A` now has inconsistent state.

> I think this approach is the most flawed and it should be avoided at all costs.
{: .prompt-warning }

## DTOs
### Using isset checks
This approach is possibly the most robust (from those I can think of) with as little boiler plate as possible. Hydrating should be easy as long as your hydratatation implementation can omit fields not set in the source parameter bag. Plus, managing annotations for API spec is also relatively easy here. 

> This is possible in PHP, but other languages like Go for example need to resort to purely dynamic data structures using maps or similar.
{: .prompt-info }

Now lets create a DTO that will replicate the data representation shown in the associative array example.
```php
readonly class UserUpdateRequest
{
    public string $email;
    public string $firstName;
    public string $lastName;
    public ?\DateTimeInterface $dateOfBirth; // to illustrate how to work with both presence and nullability
}
```

Note how the nullable field is not `null` by default. If you would do that, you would end up with the same problem as before, not being able to detect presence of nullable field correctly.

Sharp minds will notice a problem here, `isset` returns false when the member is both `null` or uninitialized.
To get around this, we can overload the magic `__isset` to leverage it's syntax (as opposed to writing `has($field)` or `hasEmail` type of checker methods).
This will add some mental overhead, but compared to the check method, I think it's worth it.

We could implement such overload like so:
```php
public function __isset($name): bool
{
    return array_key_exists($name, get_object_vars($this));
}
```
Keep in mind that this check will return `true` even when `null` value is present, so you need to further check for that, if needed.

If you are not a fan of this `__isset` overload, you can still write the check function approach, whichever you prefer more.

Now you reap benefits of DTOs, but from the usage perspective, it's still somewhat same as the array approach.
```php
    // rest of the handler code
    
    if (isset($dto->email)) {
        if (empty($email)) {
            // handle empty email as error
        }
        $user->setEmail($dto->email);
    }

    if (isset($dto->firstName)) {
        if (empty($firstName)) {
            // handle empty firstName as error
        }
        $user->setFirstName($dto->firstName);
    }

    if (isset($dto->lastName)) {
        if (empty($lastName)) {
            // handle empty lastName as error
        }
        $user->setLastName($dto->lastName);
    }

    if (isset($dto->dateOfBirth)) {
        $user->setDateOfBirth($dto->dateOfBirth);
    }

    // rest of the handler code
```

As you can see, we have pretty much improved only on maintainability part. 

If we use symfony features for validation, we're getting decently close with not a lot of syntax noise. 
This approach is probably the best for simple cases with not a lot of depth in the data structure. 
With just one level, there is not that much we can do to improve, maybe invent some nice to work with fluent API but I don't think the benefit from that
would outweigh the cost in most cases.

### Using dynamic fields
I am not the biggest fan of this approach since it's usually on the verbose side. If you go with the magic route, you have to correctly annotate the class witch each member
and it's proper type to make static analysis work even remotely. Not to mention that you have to maintain this and ensure the annotations are correct.

Better approach would be using an associative array, private constants for field names and getter/setter functions that have strict types like this example.

```php
class UserUpdateRequest
{
    private const EMAIL = 'email';
    private const FIRST_NAME = 'firstName';
    private const LAST_NAME = 'lastName';
    private const DATE_OF_BIRTH = 'dateOfBirth';

    private array $data = [];

    public function getEmail(): string
    {
        return $this->data[self::EMAIL];
    }

    public function setEmail(string $email): void
    {
        $this->data[self::EMAIL] = $email;
    }
    
    public function hasEmail(): bool
    {
        return array_key_exists($this->data[self::EMAIL], $this->data);
    }
    
    // rest of the implementations

    // our extra nullable field
    public function getDateOfBirth(): ?\DateTimeInterface
    {
        return $this->data[self::DATE_OF_BIRTH];
    }

    public function setDateOfBirth(?\DateTimeInterface $dateOfBirth): void
    {
        $this->$data[self::DATE_OF_BIRTH] = $setDateOfBirth;
    }

    public function hasDateOfBirth(): bool
    {
        return array_key_exists($this->data[self::DATE_OF_BIRTH], $this->data);
    }
}
```

Since we are trying to reduce the boilerplate needed without sacrificing type safety too much, I personally don't find this approach very useful as it 
inflates the class and the client code will be about the same as the previous approaches.

### Can we do better?
From examples set so far, my personal pick is the plain DTO paired with `isset()` checks. The code is not very verbose overall (apart from the sea of if statements, but we will try to adress this later), has strong type safety without much effort and is basically vanilla PHP (no libraries needed).
My biggest issue with this approach, however comes from frustrations you will face when you have to handle nested structures.
Suppose I have the following data structure:
```php
class Address {
    public string $street;
    public string $streetNumber;
    public string $city;
    public string $zipcode;
}
class User {
    public string $email;
    ...
    public ?Address $address;
}
```

Let's say we overriden the `__isset` to aid us with the checking and the code needs to update the values on the entity (now including the address).
It might look like this:
```php
if (isset($userDto->address)) {
    $addressDto = $userDto->address;
    if ($addressDto === null) {
        $user->removeAddress();
    } else {
        if (isset($addressDto->street)) {
            $user->address->street = $addressDto;
        }
    }
}
```
You can see how this will quickly get out of hand and it will be worse the more levels you have.

## Rust to the rescue
During my small hobby projects written in Rust I became amazed by the idea of `Option<T>`. In rust it can serve many purposes which
include null values, optional field in a struct and some others which are not super important for our purposes.
If you nest them, you can easily have a `struct` field that tracks both presence and nullability.
It's interface even has method `flatten()` which basically resolves `Option<Option<T>>` to `Option<T>` which is also helpful for this purpose.

After all PHP is not Rust and naively translating one to the other might not be very sensible. For this reason I chose to 
just represent nullability in my PHP version by using `null`, rather than using two levels of `Option<T>`.

In the next article I will go over my implementation, some improvements that I made while working with the class and I will dip into preparing
Symfony serializer component to allow hydration of fields using this class. For now you can look at the source available at the time of writing down below.

[Option.php](https://github.com/0xfrej/optional/blob/994dd23553ad141fcc4197410927ccc55a7d5600/src/Option.php)

> This version is still very WIP and final version will probably see some API changes
{: .prompt-info }

So far I used this approach in limited amount within my current workplace and it was pretty useful. Unfortunately we didn't have time to properly refactor
the biggest source of headaches, which is a feed processor consuming incremental product updates of a very massive XML structure, which was also the main 
driver to do this. However I believe that somebody, can use this idea to help them overcome some hurdles of dealing with partial hydratations of large structures.

## Alternative projects
After writing this article I discovered [php-option](https://github.com/schmittjoh/php-option/tree/master) package that I didn't know about previously.
Motivation and the library tries to solve essentially almost the same surface as I do. Some aspects of our APIs are different and the library has some nice ideas,
which I might incorporate into my version. 

However I plan to continue to work on my library, integrate it into symfony serializer and improve on the ergonomics further (current version is very heavily
derived from Rust and still needs some polishing).
