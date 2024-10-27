---
title: Handling optional fields during deserialization
description: Optional fields can be cumbersome and quite non-ergonomic to handle, even more so when you have to handle both nullability and their presence. In this small series I will try to explore an idea borrowed from Rust's Option<T> type and implement it using Symfony serializer.
author: freja
#date: # TODO: add date
categories: [PHP, Programming, Serialization, Experiments]
tags: [php, programming, serializeration]
---
Whether you're implementing DTO for REST `PATCH` action or handling complex data feed structure which utilizes partial XML for sending incremental updates,
making sure the code is maintainable and ergonomic to write can be taxing. I will mainly focus on implementing this in PHP, but essentially this can be applied
in any language. For the article I will implement a REST `PATCH` endpoint using Symfony framework with a simple nested structure to keep it clean.

## Using associative arrays or maps
More often I see optional fields (not to be confused with nullable fields) handled either using raw associative array or hash map in other languages,
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

    if (isset($data['email'])) {
        if (empty($email)) {
            // handle empty email as error
        }
        $user->setEmail($data['email']);
    }

    if (isset($data['firstName'])) {
        if (empty($firstName)) {
            // handle empty firstName as error
        }
        $user->setFirstName($data['firstName']);
    }

    if (isset($data['lastName'])) {
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
to know if it is present. Which can result in undefined behavior, say you have an entity `User` with a field `dateOfBirth` that is nullable because you don't require
your users to specify it. Now take that field in a `PATCH` request:


| **State of `dateOfBirth`** | **Request Action**                  | **Outcome**                                                                                           |
| :-------------------------- | :---------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **Not Set**                | `dateOfBirth` not included         | No change; `dateOfBirth` remains unset.                                                               |
|                             | `dateOfBirth` included with date   | Sets `dateOfBirth` on the `User` with the provided date.                                              |
| **Already Set**            | `dateOfBirth` not included         | **Option 1**: Keep the existing `dateOfBirth`. (prohibits user from ever removing the date from their account) <br> **Option 2**: Clear (set to `null`).              |
|                             | `dateOfBirth` included with date   | Updates `dateOfBirth` on the `User` with the new date from the request.                               |

Now, you can think that this is just a minor issue, frontend can handle it by always sending the value from the entity even if it was not changed, thus allowing users
to also unset the `dateOfBirth` field. However, this not only breaks the conventions tied to the `PATCH` method, it will come with even more issues when the caller is 
not in your hands. 

### Let's think of a data feed for a second
This feed has many, many fields. One of them is some very important value that is nullable. Now, you will receive
updates from this feed in partial form. Meaning you have to now handle presence as well. 

Suppose that one day entity `A` had an update in their database and it changed
from something to `null`. The update will also have the field set to `null`, but since our code does not know difference between field being present, and we most likely don't 
want to reset it each time you get an update, meaning that now this field didn't get set to `null` in our system after the update and the entity `A` now has inconsistent state.

> I think this approach is the most flawed and it should be avoided at all costs.
{: .prompt-warning }

## DTOs
### using isset checks
This approach is possibly the most robust (from those I can think of) with as little boiler plate as possible. Hydrating should be easy as long as your hydratatation implementation can omit hydrating fields not set in the source parameter bag. Plus, managing annotations for API spec is also relatively easy here. Now lets create a DTO that will replicate the data representation shown in the associative array example.

> This is possible in PHP, but other languages like Go for example need to resort to purely dynamic data structures using maps or similar.
{: .prompt-info }

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

Now you reap benefits of DTOs, but from the syntax perspective, it's still somewhat same as the array approach.
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

As you can see, we have pretty much improved only on maintainability part. You can also omit the validation part as that can be offloaded in Symfony using attributes on the DTO.
If we use attributes to validate, we're getting decently close with not a lot of syntax noise. This approach is probably the best for simple cases with not a lot of depth
in the data structure. With just one level, there is not that much we can do to improve, maybe invent some nice to work with fluid API but I don't think the benefit from that
would outweigh the cost in most cases.

### using dynamic fields
Now, purely dynamic DTOs are not very useful as in principle they will function as the array approach. I can think of two flavors of this approach 
and I will talk about pros and cons of each in a second. This approach is decent, fairly easy to implement 
TODO: write this part and an example getters vs magic? or just use getters and mention magic
TODO: think if there is some more to talk about and start a new draft with the multi level version of the dto and show how cumbersome it can become (multi levels, collections of optionals)
