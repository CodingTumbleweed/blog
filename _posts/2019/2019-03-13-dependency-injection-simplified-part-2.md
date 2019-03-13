---
layout: single
title: "Dependency Injection Simplified - Unity for DI"
date: 2019-03-13 -2200
classes: wide
categories: [design_pattern]
tags:
- Dependency Injection
- Code
- C-Sharp
- Unity
---

In [Part 1](/design_pattern/dependency-injection-simplified-part-1/) of this series, we refactored the code for serialization to make it decoupled using dependency injection. This saves us a lot for work in future when there's a change request. In this part, we are going to do the same using **Unity**, a popular framework for dependency injection.  Unity framework implements dependency injection in 2 parts:

1. **Registration:**

   First you register an interface with it's desired associated class. This is how you tell Unity which object of a class to instantiate for a particular interface. Registration is typcially done at the start of program.

2. **Resolution: **
  This is the part where you resolve an  interface to get an object of class associated with it. Note that you can associate an interface with either a single class or multiple classes. In latter case, you need to give all associations a name to identify them. we will use named registration in our example below.

Now lets start by installing Unity through nuget package manager console:

```csharp
Install-Package Unity
```

In out project, lets make an interface for serialization:

```csharp
interface ISerialization
{
    //UserPreference is data model class
    void WriteFile(UserPreference obj, string filePath, bool append = false);
    UserPreference ReadFile(string filePath);
}
```



And then write an implementation for both XML and Binary serialization using this interface:

```csharp
class XmlSerialization : ISerialization
{
    /// <summary>
    /// Read Xml File and returns it as UserPreference object
    /// </summary>
    /// <param name="filePath">FileName including file path</param>
    public UserPreference ReadFile(string filePath)
    {
        using (TextReader reader = new StreamReader(filePath))
        {
            var serializer = new XmlSerializer(typeof(UserPreference));
            return (UserPreference)serializer.Deserialize(reader);
        }
    }
    
    /// <summary>
    /// Writes UserPreference object to Xml File
    /// </summary>
    /// <param name="obj">Object to write</param>
    /// <param name="filePath">FileName including file path</param>
    /// <param name="append">Append or Overwrite to File</param>
    public void WriteFile(UserPreference obj, string filePath, bool append = false)
    {
        using (TextWriter writer = new StreamWriter(filePath, append))
        {
            var serializer = new XmlSerializer(typeof(UserPreference));
            serializer.Serialize(writer, obj);
        }
    }
}
```

```csharp
class BinarySerialization : ISerialization
{
	/// <summary>
    /// Read Binary File and returns it as UserPreference object
    /// </summary>
    /// <param name="filePath">FileName including file path</param>
    public UserPreference ReadFile(string filePath)
    {
        using (Stream stream = File.Open(filePath, FileMode.Open))
        {
            var binaryFormatter = new BinaryFormatter();
            return (UserPreference)binaryFormatter.Deserialize(stream);
        }
    }

	/// <summary>
    /// Writes UserPreference object as Binary File
    /// </summary>
    /// <param name="obj">Object to write</param>
    /// <param name="filePath">FileName including file path</param>
    /// <param name="append">Append or Overwrite to File</param>
    public void WriteFile(UserPreference obj, string filePath, bool append = false)
    {
        using (Stream stream = File.Open(filePath, append ? FileMode.Append : FileMode.Create))
        {
            new BinaryFormatter().Serialize(stream, obj);
        }
    }
}
```



Till now, it's the same code we did last time. Now we are going to make a **Factory** class with methods ***RegisterInterfaces()*** and ***Resolve(name)***:




```csharp
class Factory
{
    internal IUnityContainer Container { get; private set; }

    public Factory() : this(new UnityContainer()) { }

    private Factory(IUnityContainer _container)
    {
        Container = _container;
    }

    /// <summary>
    /// Register all interfaces and their Types
    /// </summary>
    internal void RegisterInterfaces()
    {
        //ContainerControlledLifetimeManager creates a singleton object on first call and then returns the same object on subsequent calls 
        Container.RegisterType<ISerialization, XmlSerialization>("XML", new ContainerControlledLifetimeManager());
        Container.RegisterType<ISerialization, BinarySerialization>("BINARY", new ContainerControlledLifetimeManager());
    }

    /// <summary>
    /// Creates object of type T given an interface
    /// </summary>
    /// <param name="name">Name if there's a named registration</param>
    /// <returns></returns>
    internal T Resolve<T>(string name = null) where T : class
    {
        return string.IsNullOrEmpty(name) ? Container.Resolve<T>() : Container.Resolve<T>(name);
    }

}
```



With this single **Factory** class, we are almost done. Now you just need to call ***RegisterInterfaces()*** at start of your main class:


```csharp
Factory objFactory = new Factory();
objFactory.RegisterInterfaces();
```



And then whenever we want to serialize/deserialize  a file, call the **resolve** function to get *serializer* object:

```csharp
var serializer = objFactory.Resolve<ISerialization>("XML");
```


And use this *serializer* object to write to and read from this file:

```csharp
UserPreference Preference = new UserPreference();
serializer.WriteFile(Preference, "UserPrefsFile");
```

```csharp
Preference = serializer.ReadFile("UserPrefsFile");
```



That should get your serialization up and running!