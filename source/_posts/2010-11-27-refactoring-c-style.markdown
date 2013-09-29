---
layout: post
title: "Refactoring C# Style"
date: 2010-11-27 09:51
comments: true
categories: 
---

Take a look a this function.

``` c#
[UnitOfWork]
public virtual void Upload(DocumentDto dto)
{
    var entity = new Document().Merge(dto);
 
    using (var stream = new MemoryStream(dto.Data))
    {
        repository.Save(entity, stream);
    }
}
```

After doing some performance profiling, it quickly popped up as the top offender. Why? Because it’s holding a database transaction open while saving a file. In order to fix it, we have to ditch our `UnitOfWork` aspect and implement a finer-grained transaction. Basically what needs to happen is saving the entity and saving the file need to be separate operations so we can commit the transaction as soon as the entity is saved. And since saving the file could fail, we might have to clean up an orphaned file entity.

``` c#
public virtual void Upload(DocumentDto dto)
{
    var entity = new Document().Merge(dto);
 
    SaveEntity(entity);
 
    try
    {
        SaveFile(entity, dto.Data);
    }
    catch
    {
        TryDeleteEntity(entity);
        throw;
    }
}
 
private void SaveEntity(Document entity)
{
    unitOfWork.Start();
    try
    {
        repository.Save(entity);
        unitOfWork.Commit();
    }
    catch
    {
        unitOfWork.Rollback();
        throw;
    }
}
 
private void SaveFile(Document entity, byte[] data)
{
    using (var stream = new MemoryStream(data))
    {
        repository.Save(entity, stream);
    }
}
 
private void TryDeleteEntity(Document entity)
{
    unitOfWork.Start();
    try
    {
        repository.Delete(entity);
        unitOfWork.Commit();
    }
    catch
    {
        unitOfWork.Rollback();
    }
}
```

That wasn’t too bad except that the number of lines of code exploded and we have a few private methods in our service that only deal with plumbing. It would be nice to push them into the framework. Since C# is awesome, we can use a combination of delegates and extension methods to do that.

``` c#
public virtual void Upload(DocumentDto dto)
{
    var entity = new Document().Merge(dto);
 
    unitOfWork.Execute(() => repository.Save(entity));
 
    try
    {
        repository.SaveFile(entity, dto.Data);
    }
    catch
    {
        unitOfWork.TryExecute(() => repository.Delete(entity));
        throw;
    }
}
 
public static class DocumentRepositoryExtensions
{
    public static void SaveFile(this IDocumentRepository repository, Document document, byte[] data)
    {
        using (var stream = new MemoryStream(data))
        {
            repository.SaveFile(document, stream);
        }
    }
}
 
public static class UnitOfWorkExtensions
{
    public static void Execute(this IUnitOfWork unitOfWork, Action action)
    {
        unitOfWork.Start();
        try
        {
            action.Invoke();
            unitOfWork.Commit();
        }
        catch
        {
            unitOfWork.Rollback();
            throw;
        }
    }
 
    public static void TryExecute(this IUnitOfWork unitOfWork, Action action)
    {
        unitOfWork.Start();
        try
        {
            action.Invoke();
            unitOfWork.Commit();
        }
        catch
        {
            unitOfWork.Rollback();
        }
    }
}
```

Now we can move the extension methods into the framework and out of our service. In a less awesome language we could define these convenience methods on the `IUnitOfWork` interface and implement them in an abstract base class, but inheritance is evil and it’s a tradeoff we don’t have to make in C#.