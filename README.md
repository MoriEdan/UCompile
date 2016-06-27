#Runtime C# script engine for Unity3D

##Compile and execute C# code inside Unity3D scene at runtime!

UCompile is a system for compiling and executing strings with C# code inside Unity3D scenes at runtime. You can use it to allow players of your Unity3D game to modify your game with C# code, or as REPL engine, possibilities are restricted by only your imagination! For usage examples see <a href="#usage examples">usage examples chapter</a>. Works in editor and in build.

Tested in Unity3D on:
* Windows 8,8.1,10.

For now project works only on Windows.

If you have found a bug, create an issue on the [github page](https://github.com/SoapCode/UCompile), or a pull request if you have a fix / extension. If you have a question, you can email me at soapcode24@gmail.com. 

##Main Features
* Compilation and execution of classless and methodless C# code
* C# classes compilation
* Control on namespaces exposed to the code to restrict its access of resources
* C# code compilation and execution in separate AppDomain to control memory consumption
* Coroutine compilation and execution
* Works in build as well as in editor
* Example project with console-like GUI interface to test it all out

##Installation

You can simply download/clone this repo and copy UCompile folder to your Unity3D project.

## <a id="How it works"></a>How it works?
The main principle behind the scenes is simple: take string with C# code, compile it and produce Assembly representing this code, load it into current AppDomain. 

<img src="DocumentationMisc/CompilationScheme.png?raw=true" alt="compilation scheme" width="940px" height="454px"/>

Now this code is officially a part of your application, as long as AppDomain it is loaded into stays loaded. 

There are 2 ways of how UCompile allows you to interact with these assemblies:

1. Compiling classes. You can compile your custom classes, and make them a part of your Unity application. These classes will be able to use any functionality you've decided to expose to it, from assemblies, which are loaded right now in main Unity AppDomain, including assemblies with classes which you dynamically compiled with UCompile earlier.
2. Compiling and executing methodless code. You can compile plain methodless code, using the same exposure restriction system. Imagine that you put your code in some kind of a "Main()" method, that you can execute at any moment, and this code can use classes, that you have already dynamically compiled with the first way.

It's going to make sense soon, I promise! Let's look at some examples:

<a id="Methodless compilation"></a>**1. Methodless code compilation and execution**

Here let's create an empty scene, and add an empty GameObject to it with following script attached.

```csharp
using UnityEngine;
using UCompile;

public class CompileCodeExample : MonoBehaviour 
{

    // Update is called once per frame
    void Update ()
    {
	
        if(Input.GetKeyDown(KeyCode.Space))
        {
            CSScriptEngine engine = new CSScriptEngine();

            engine.AddUsings("using UnityEngine;");

            IScript result = engine.CompileCode("GameObject.CreatePrimitive(PrimitiveType.Cube);");
            result.Execute();
        }

     }
}
```

On space press, we create new instance of the main class you need to worry about in UCompile, CSScriptEngine. Then, via AddUsings method, we, sort of, add using directive to the code, that we are going to compile. So UnityEngine namespace classes are now available for it to use. After that, we invoke CompileCode method of CSScriptEngine, passing string with code as a parameter, and we save the resulting IScript type. Behind the scenes, string with methodless code will be wrapped in a method called Execute and a class, and this class implements IScript interface. After compilation, instance of this class is returned by CompileCode as IScript object. Then you can invoke this objects Execute method, to execute code, you've just compiled. This is the "Main()" method we discussed earlier.

So basically what happens here - your code gets wrapped in a method and a class, then this class is compiled and instance of this placeholder class is returned by CompileCode as interface object. Then method containing our code, called Execute, is invoked, and thats how our code gets executed. Try it, and you'll see a cube appear! 

This way you can interact with your Unity scene via code while it's running. If you want some kind of REPL console functionality in your scene, and you don't need to dynamically add more functionality, by compiling classes, that's could be all you need from UCompile. But we can do more.

<a id="Class compilation"></a>**2. Class compilation**

What if we want not only to create cube in our scene, but also make it change color on button press? So lets say code of this changing color MonoBehaviour goes like this:

```csharp
public class ColourChanger : MonoBehaviour
{
    void Update()
    {
        if (Input.GetKeyDown(KeyCode.C))
        {
            this.gameObject.GetComponent<MeshRenderer>().material.color = new Color(Random.value, Random.value, Random.value);
        }
    }
}
```

Of course we can write changing color MonoBehaviour and compile it with the rest of scripts at compilation time of our Unity application, then create cube and attach this MonoBehaviour to it, using CompileCode method as we described above. Code of MonoBehaviour doing that would look like this:

```csharp
using UnityEngine;
using UCompile;

public class CompileClassExample : MonoBehaviour 
{

    // Update is called once per frame
    void Update ()
    {
	
        if(Input.GetKeyDown(KeyCode.Space))
        {
            CSScriptEngine engine = new CSScriptEngine();

            engine.AddUsings("using UnityEngine;");

            IScript result = engine.CompileCode(@"
            					   GameObject cube = GameObject.CreatePrimitive(PrimitiveType.Cube);
            					   cube.AddComponent<ColourChanger>();
            					");
            result.Execute();
        }

     }
}
```

But what if we want to create this ColorChanger MonoBehaviour dynamically at runtime? With some slight modifications of above code, we can do that!

```csharp
using UnityEngine;
using UCompile;

public class NewBehaviourScript : MonoBehaviour {

    // Update is called once per frame
    void Update ()
    {
	
        if(Input.GetKeyDown(KeyCode.Space))
        {
            CSScriptEngine engine = new CSScriptEngine();

            engine.AddUsings("using UnityEngine;");

            string typeCode = @"
                
                                public class ColourChanger : MonoBehaviour
                                {
                                    void Update()
                                    {
                                        if (Input.GetKeyDown(KeyCode.C))
                                        {
                                            this.gameObject.GetComponent<MeshRenderer>().material.color = new Color(Random.value, Random.value, Random.value);
                                        }
                                    }
                                }

                              ";

            engine.CompileType("ColorChanger", typeCode);

            IScript result = engine.CompileCode(@"
                                                 GameObject cube = GameObject.CreatePrimitive(PrimitiveType.Cube);
                                                 cube.AddComponent<ColourChanger>();
                                               ");
            result.Execute();
        }

	}
}
```
Here we save our ColorChanger MonoBehaviour code in typeCode variable, then we pass it to CompileType(string typeID, string code) method, with "ColorChanger" as typeID. CompileType method associates "ColorChanger" string ID with typeCode, so whenever you want to change it, you need to call CompileType with the same ID you initially compiled it and your new typeCode. So every type you compile should have its unique ID, otherwise it will be treated as already existing type, and calling CompileType with typeID of this type will result into its typeCode changed to the one you passed. 

So this CompileType call sort of adds this type to the system, and from now on, code passed to CompilCode or CompileType methods will have access to this type and can perform operations on it. You can see, that now, after we compiled this type and "added it to the system", using our old friend CompileCode method we can do whatever we want with it, for example attaching it to our cube GameObject! Try it, and you'll have a cube in your scene changing color every time you press C.

Be aware, that every time you change typeCode of already existing type, previous version of this type is discarded, and only the last compiled version will be available to use for the code passed to CompileCode and CompileType methods. 

**Summing it up in few words:** you can compile methodless code with CompileCode method, and compile classes with CompileType method. You can control what this code can access by adding using derictives via AddUsings method. All classes you compiled with CompileType will be accessible constantly.

##UCompile structure

<img src="DocumentationMisc/UcompileStructureScheme.png?raw=true" alt="ucompile structure scheme" width="1035px" height="240px"/>

Here I'll give you a brief overview of the system structure, for more details you can dig into the code, there's not much and it's all heavily commented.

3 main building blocks of UCompile are: MonoEvaluator.cs, CSScriptEngine.cs and CSScriptengineRemote.cs.

###Module MonoEvaluator.cs

MonoEvaluator is the main class of this module, it's job is to encapsulate instance of Mono.Csharp.Evaluator(the chosen way to dynamically compile code in UCompile), feed code strings to it with method CompileCode, handle compilation errors and warnings, save them in special container-class CompilerOutput. If errors occured during compilation, MonoEvaluator will throw a custom exception CompilerException, with information about all errors and warnings. Also MonoEvaluator contains property CompilationOutput, allowing you to get information about last compilation regardless of if it failed or not. 

Method ReferenceAssemblies of class MonoEvaluator "binds" assemblies to Mono.Csharp.Evaluator instance under the hood of MonoEvaluator, which allows it to expose these assemblies functionality to code that is to be compiled by Mono.Csharp.Evaluator. You still need to include using directives in your code though, but that's where using directives control system comes into play of CSScriptEngine class, which we will discuss later.

###Module CSScriptEngine.cs

This module contains class CSScriptEngine, the main class of the whole UCompile system. It uses wrapped MonoEvaluator class instance to perform compilation. You're supposed to interact with this class first and foremost. Some of its methods:

**1. public IScript CompileCode(string code = "")**

This method compiles methodless code, and returns IScript object. You can execute your methodless code by invoking Execute method of IScript object. Returns null, if compilation failed. For examples see  <a href="#Methodless compilation">Methodless code compilation and execution</a> in <a href="#How it works">How it works</a> chapter.

**2. public Type CompileType(string typeID, string code)**

This method compiles class code(that is string with code describing class) saving reference to it by using user provided typeID. Once you compiled a type, it's now in system, and if you want to change its code - use CompileType with the same typeID you initially provided for this type, and your new type code. Now, when you use this class in your dynamic code only the last compiled version will be used. Examples: <a href="#Class compilation">Class compilation</a>.

**3. public void RemoveTypes(params string[] typeIDs)**

Removes type from the system given its typeID. After that you can't use this type in your dynamic code.

**4. public IEnumerable CompileCoroutine(string coroutineCode = "")**

Did I mention, we also can compile coroutines? ;-) Same principles as with CompileCode apply, with few additions. Apart from that you need to place yield return somewhere in your coroutine code, this method returns IEnumerable object. Returning this instead of IEnumerator allows us to "rewind" coroutine every time we invoke GetEnumerator on IEnumerable object, what is pretty handy. You can look at some usage examples in <a href="#cheatsheet">CheatSheet</a>.

**5. public void AddOnCompilationSucceededHandler(Action<CompilerOutput> onCompilationSucceededHandler)**

This method as well as its relatives with similar signature, allows you to, roughly speaking, "subscribe" and "unsubscribe" "event handlers" to the compilation succeded and compilation failed "events". Depending on whether last compilation succeded or failed, CSScriptEngine will execute related delegate, passing CompilerOutput instance with information about warnings and errors. You can subscribe to this delegate with your Action<CompilerOutput> method, to do whatever you want. Fo example, you can transmit errors and warnings to some kind of output window. See examples <a href="#cheatsheet">here</a>.

**6. public void AddUsings(string usings)**

This method allows you to add one or multiple using directives to system, once added these usings apply to all your dynamic code, until you remove them. Thats how you control what is visible and accessible to dynamic code. Also it automatically references assemblies related to usings. Beware, throws exceptions! If you're not sure about input, probably want to handle them.

**7. public void RemoveUsings(string usings)**

Removes usings from system. Throws exceptions as well!

**8. public void Reset()**

Removes all usings and all references to previously compiled types from CSScriptEngine.

###Module CSScriptEngineRemote.cs

Every time you compile code using CSScriptEngine, assembly is created and loaded to current AppDomain. Problem is, you can't unload this assembly, without unloading whole AppDomain. So, if you have an infinite amount of code to compile, you'll need to unload your  AppDomain at some point to free memory occupied with unused assemblies. In order to provide option to avoid this limitation, CSScriptEngineRemote.cs module was created. CSScriptEngineRemote is the main class of this module, what it does is it creates another separate AppDomain and loads CSScriptEngine instance to it, then sends signals to it to perform operations we're already familiar with. This way, code compilation and, therefore, assembly creation and loading occurs in separate AppDomain. Which at any point we can unload with all dynamic assemblies, and free memory, without unloading our main AppDomain. CSScriptEngineRemote is designed to work similar to CSScriptEngine, but it has some nuances.

**1. public void LoadDomain(string name = "")**

Creates new AppDomain inside Unity application. That's where our remote CSScriptEngine instance will go. You need to call LoadDomain before calling any other method in CSScriptEngineRemote.

**2. public void UnloadDomain()**

Unloads remote AppDomain. CSScriptEngineRemote implements IDisposable interface, and this method will be called in Dispose() 
automatically, if you didn't call it before manually. Be aware, you can't leave remote AppDomain hanging when you're done working with CSSCriptEngineRemote, you have to call UnloadDomain or Dispose if you called LoadDomain previously.

**3. public void CompileCode(string code)**

As you can see, here CompileCode doesn't return IScript object, but rather saves it internally. Other than that, it works in a similar manner with CSScriptEngine. 

**4. public void ExecuteLastCompiledCode()**

This method calls Execute function of last compiled, saved internally, IScript object. If with CSScriptEngine we get to do it manually, here this function does it for us.

**5. public void CompileType(string id, string code)**

CompileType also now doesn't return Type object. And it can't compile MonoBehaviours, oh by the way, never compile MonoBehaviours with CSScriptEngineRemote.CompileType(), it may appear to work fine at first, but will inevitably lead to strange behaviours, bugs and crashes. Plain C# classes work fine.

Other methods work pretty similar to CSScriptEngine. So, your best bet with CSScriptEngineRemote is to compile plain C# classes and methodless code, you can't compile coroutines, every dynamically created with code in CSScriptEngineRemote GameObject must be destroyed before unloading AppDomain. Moreover messing with AppDomains in Unity can sometimes lead to a bunch of strange and unexpected bugs, so be aware before using CSScriptEngineRemote. That's the price you pay for ability to free memory from dynamic assemblies at any time during runtime. 

Examples: <a href="#cheatsheet">CheatSheet</a>.
