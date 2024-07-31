---
layout:   post
title:    "The human typewriter, or why optimizing for typing is short-sighted"
tags:     programming c
category: Programming
---

## Intro
Quick caveat before we start: This post can definitely be considered a "religious" opinion piece - this is purely my opinion, and you're free to disagree with me.

[In fact, please do!](https://mastodon.gamedev.place/@FelixK15)

### Don't optimize for typing

What I want to touch on in this post is the increased usage of `auto` and template type deduction that I'm seeing in newer codebases.
I'm not a big fan of either since they unnecessarily obfuscate the codebase in favor for...being able to faster type code...? ¯\\\_(ツ)_/¯  

Optimizing your coding experience for typing is, in my opinion, the worst axis to optimize for when working on a codebase. `auto` lovers will say that the IDE will resolve the types for you if you really care, but would someone *please* think of our poor Vim & Emacs users?

In my opinion, it puts unnecessary mental strain on the developer reading the code since it's not clear at first glance what the `auto` type actually resolves to. Again, the IDE will probably help you here but...just..."why" in the first place?

IMHO code should be optimized for ~~reading~~ ~~debugging~~ *navigation*. You want a codebase that's easy to navigate - a codebase that's easy to navigate is, in my experience, likewise easy to debug and easy to read. Omitting the use of `auto` (and template type deducation) is just a subset of the things I'm seeing in codebases that make them hard to navigate.


### Code example
What does that mean in practice?
Let's consider this arbitrary code snippet that I just pulled out of thin air:
```cpp
struct database_arguments;
{
    const char* pDatabasePath;
    const char* pQuery;
};

template <typename T>
class result
{
public:
  enum class result_type
  {
    success,
    out_of_memory,
    invalid_args,
    connection_refused,
    //...
  };

  T getValue();
  bool isFailure();

private:
  result_type resultType;
  T value;
};

void updateDataInDatabase(const database_arguments* pArgs)
{
    //IDatabase has a virtual function `createQuery()`, `readData()`, `close()`, `writeData()`
    IDatabase* pDatabase = openDatabase(pArgs->pDatabasePath);    

    //Result is a template type that contains a known result type (success,failure), which can be checked by `isFailure()`, and the actual result value from the function   which can be retrieved by `getResult()`
    result queryResult = pDatabase->createQuery(pArgs->pQuery);
    if(queryResult.isFailure())
    {
        pDatabase->close();
        return;
    }   

    auto dataSet = pDatabase->readData(queryResult.getValue());
    auto data = dataSet.getData();
    auto dataSize = dataSet.getDataSize();  

    updateLocalData(data, dataSize); 

    pDatabase->writeData(queryResult.getValue(), data, dataSize);
    pDatabase->close();
    return;
}
```

Just looking at this snippet:
- It's not clear what type `dataSet`, `data` & `dataSize` are.
- It's not clear what unit the value in `getDataSize()` is - is it bytes, kilobytes maybe even megabytes?
- `IDatabase` is polymorphic; what actual implementation function is being called when calling `IDatabase::createQuery()`, `IDatabase::writeData()` & `IDatabase::readData()`?
- `Result` is templated; what is the result value of `IDatabase::createQuery()`?
- Is it safe to modify the data returned by `dataSet.getData()`?

> *Note*: The use of a templated Result type is IMHO perfectly valid here.

On the surface, this seems super nitpicky, but believe me - once you've read enough codebases that make heavy use of this, you get real tired *real* fast.
Maybe a better approach would be this:

```cpp
void updateDataInDatabase(const database_arguments* pArgs)
{
    if(isSQLDatabasePath(pArgs->pDatabasePath))
    {
        SQLDatabase* pSqlDatabase = openSqlDatabase(pArgs->pDatabasePath);
        updateSQLDatabaseEntry(pSqlDatabase, pArgs);
    }
    else
    {
        //...
    }
}

void updateSQLDatabaseEntry(SqlDatabase* pDatabase, const database_arguments* pArgs)
{
    result<DatabaseQuery> queryResult = createSQLDatabaseQuery(pArgs->pQuery);
    if(queryResult.isFailure())
    {
        pDatabase->close();
        return;
    }

    DatabaseDataSet dataSet = pDatabase->readData(queryResult.getValue());
    uint8_t* pData = dataSet.getDataShadowCopy();
    const uint64_t dataSizeInBytes = dataSet.getDataSizeInBytes();

    updateLocalData(pData, dataSizeInBytes);

    pDatabase->writeData(queryResult.getValue(), pData, dataSizeInBytes);
    pDatabase->close();
    return;
}
```
Everything that was unclear by the previous example has been made explicit here.
We know what `IDatabase` implementation we're working with, we know what the unit of the size of the data is, and we know if it's safe to modify the data from the `dataSet`.

### Outro

> *Note*: Yes, I know that the example could've been rewritten a million different ways to weaken my arguments here, but c'mon - nobody likes a party pooper, I hope that you know what I'm trying to say here :)

The next time you submit a pull request or release a piece of code try to put yourself in the shoes of someone who has no patience, your personal address, a lot of time and only notepad as a code editor.
