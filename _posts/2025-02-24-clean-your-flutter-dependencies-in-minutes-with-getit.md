---
layout: post
title: "Clean Your Flutter Dependencies in Minutes with GetIt"
category: tutorials
tags: flutter dart tutorial plugin "dependency injection" 
medium: "https://medium.com/easy-flutter/clean-your-flutter-dependencies-in-minutes-with-getit-dc510937a201"
---

Am I the only one who ***hates*** this sight:

```dart
class App extends StatelessWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context) {
    final client = HttpClient();
    final prefs = await SharedPreferences();
    final depA = DependencyA();
    final depB = DependencyB();
    final depC = DependencyC();

    return MultiRepositoryProvider(
      providers: [
        RepositoryProvider<RepositoryA>(create: (_) => RepositoryAImpl(client, prefs)),
        RepositoryProvider<RepositoryB>(create: (_) => RepositoryBImpl(depA)),
        RepositoryProvider<RepositoryC>(create: (_) => RepositoryCImpl(depB)),
        RepositoryProvider<RepositoryD>(create: (_) => RepositoryDImpl(client)),
        RepositoryProvider<RepositoryE>(create: (_) => RepositoryEImpl(depC)),
      ],
      child: MultiBlocProvider(
        providers: [
          BlocProvider<BlocA>(
            create:
                (_) =>
                    BlocA(context.read<RepositoryA>())..add(InitEvent()),
          ),
          BlocProvider<BlocB>(
            create:
                (_) =>
                    BlocB(context.read<RepositoryB>())..add(InitEvent()),
          ),
          BlocProvider<BlocC>(
            create: (_) => BlocC(context.read<RepositoryC>()),
          ),
          BlocProvider<BlocD>(
            create:
                (_) => BlocD(
                  context.read<RepositoryD>(),
                  context.read<RepositoryE>(),
                ),
          ),
        ],
        child: MaterialApp(

// ...the rest of this miserable app
```

Managing dependencies efficiently is crucial, especially as application scopes grow. The [get_it](https://pub.dev/packages/get_it) package can simplify dependency injection by acting as a service locator, allowing developers to register and retrieve instances easily. We’ll use this to establish **all of our dependency relationships in one easy-to-read file**.

## Installing GetIt 
Start by adding the package to your `pubspec.yaml`:

```yaml
// pubspec.yaml

dependencies:
  get_it: ^7.6.7
```

## Registering Dependencies
We can start by creating what I call an injection container, where we register services or repositories in a single setup function:

```dart
// injection_container.dart

final di = GetIt.I;

Future<void> setupDependencyInjection() async {
  di
    ..registerSingleton<HttpClient>(HttpClient())
    ..registerSingleton<SharedPreferences>(await SharedPreferences.getInstance())
    ..registerSingleton<RepositoryA>(RepositoryA(di()))
    ..registerFactory<BlocA>(BlocA(di())
}
```

Immediately, we make the **GetIt** singleton instance a global variable: `di`. We do this exclusively to invoke the dependencies we will register, later in our `BlocProviders` and `MultiBlocProviders`.

To register our dependencies, the **GetIt** instance provides a few methods:

`registerSingleton<T>()` – Creates a single instance of the class and reuses it.
`registerLazySingleton<T>()` – Lazily creates a single instance of the class when first accessed and reuses it.
`registerFactory<T>()` – Creates a new instance every time it’s requested.
Note: Because **GetIt** keeps an active record of all instances registered and their type, it can infer which instance to produce when called as a constructor argument. So as long as dependency registrations are properly ordered, simply calling `di()` as an argument is enough.

Now let’s create the same dependency chain from the first example:

```dart
// injection_container.dart

final di = GetIt.I;

Future<void> setupDependencyInjection() async {
  di
    // Services
    ..registerSingleton<HttpClient>(HttpClient())
    ..registerSingleton<SharedPreferences>(await SharedPreferences.getInstance())
    ..registerSingleton<DependencyA>(DependencyA())
    ..registerSingleton<DependencyB>(DependencyB())
    ..registerSingleton<DependencyC>(DependencyC())

    // Repositories
    ..registerSingleton<RepositoryA>(RepositoryAImpl(di(), di()))
    ..registerSingleton<RepositoryB>(RepositoryBImpl(di()))
    ..registerSingleton<RepositoryC>(RepositoryCImpl(di()))
    ..registerSingleton<RepositoryD>(RepositoryDImpl(di()))
    ..registerSingleton<RepositoryE>(RepositoryEImpl(di()))

    // Blocs
    ..registerFactory<BlocA>(BlocA(di())
    ..registerFactory<BlocB>(BlocB(di())
    ..registerFactory<BlocC>(BlocC(di())
    ..registerFactory<BlocD>(BlocD(di(), di())
}
```

We then only need to call this function once before `runApp()`:

```dart
// main.dart

void main() {
  await setupDependencyInjection();
  runApp(MyApp());
}
```

## Accessing Dependencies
Lastly, we now only need to provide our registered Bloc instances in the `MultiBlocProvider`:

```dart
// app.dart

class App extends StatelessWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context) {
    return MultiBlocProvider(
      providers: [
        BlocProvider<BlocA>(create: (_) => di())                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     ,
        BlocProvider<BlocB>(create: (_) => di()),
        BlocProvider<BlocC>(create: (_) => di()),
        BlocProvider<BlocD>(create: (_) => di()),
        BlocProvider<BlocE>(create: (_) => di()),
      ],
      child: MaterialApp(

// ...the rest of this less miserable app
```

…much cleaner and easier to manage.

The best part of this approach: Since we register our Blocs as “factories”, those provided to specific screens/widgets will be **destroyed when their routes are popped**. This avoids unused Blocs and their dependencies unnecessarily using memory.

## Conclusion
The **GetIt** package was intended as a global service locater, but for developers who **prefer explicit dependency injection**, it can act as a **dependency relationship manager**. So whenever dependencies and their relationships change, a developer doesn’t have to scour their codebase for references.
