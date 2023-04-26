---
title: "Workmanager Flutter"
categories:
  - blog
author_profile: false
---

# How to run Flutter in the background?



**WorkManager** provides an easy-to-use API where you can define:

- If a background job should run **once**, or **periodically**
- Apply multiple **constraints** like for example if a task needs internet connection, or if the battery should be     fully charged, and so much more.
- An **interval** of how many times a job should run.

Add dependencies in pubspec yaml:

```dart
dependencies:
  workmanager: ^0.5.1
```

Let’s setup for android:

Make sure that your kotlin_version to `1.5.0`
 or greater:

```dart
buildscript {
    ext.kotlin_version = '1.5.+'
    repositories {
        google()
        mavenCentral()
    }
```

Check if you have the following in your `AndroidManifest.xml`
 file.

```dart
<meta-data
    android:name="flutterEmbedding"
    android:value="2" />
```

## How to debug my background job?

However to facilitate debugging, the plugin provides an `isInDebugMode` flag when initializing the plugin: `Workmanager().initialize(callbackDispatcher, isInDebugMode: true)`.

In below example I am doing in  you can see it there.

Once this flag is enabled you will receive a notification whenever a background task was triggered.This way you can keep track whether that task ran successfully or not.

<!-- Let’s setup for ios:
**Prerequisites**

This plugin is compatible with **Swift 4.2**
 and up. Make sure you are using **Xcode 10.3**
 or higher and have set your minimum deployment target to **iOS 10**
 or higher by defining a platform version in your podfile: `platform :ios, '10.0'`

In xcode runner signing and capabilites  background mode and check background processing which will ad UIBckgroundModes key to Info.plist

```
<key>UIBackgroundModes</key>
<array>
	<string>processing</string>
</array>
``` -->

<!-- Will continue on android then come to on ios part: -->

## Example Scenario
Let me give you overview what I am trying to do in this example:
In my app there you can order even if you are offline.Basically the scenario i am assuming is 
- **User** opens the app he **orders** something which will be in **cart**.
- When user orders We have to order all the cart items he had loaded in cart.
- Then user can terminate app from background.

The main goal here is to if he even **terminates app** we gonna place order which means we gonna do **api call**. 
Let's jump to that part.

Here I will show you  registering one task : 

In my cart service you can see I am checking the internet connection if it has i am gonna do normal order.
Otherwise I am gonna 

```dart
Future<Either<Failure, void>> placeOrder(Iterable<CartItem> cartItems, String orgId) async {
    try {
      bool hasConnection = await _checkConnection();
      if (hasConnection) {
        await cartRepository.placeOrder(cartItems, orgId);
        await cartRepository.clear();
        return const Right(null);
      } else {
      
        final sendingData = jsonEncode(cartItems.toList());

        await Workmanager().registerOneOffTask(
          '1',
          mySimpleTask,
          backoffPolicy: BackoffPolicy.linear,
          backoffPolicyDelay: const Duration(seconds: 20),
          inputData: <String, dynamic>{
            'string': orgId,
            'array': sendingData,
          },
        );

        return const Right(null);
      }
    } catch (e) {
      return Left(Failure(message: e.toString()));
    }
  }
```

**Note:**  Here you can’t send cart items list.That is why I am using **jsonEncode** function to convert it into string of json. If you don't use this you  will get exception like:

```dart
throw Exception(
              "argument $key has wrong type. WorkManager supports only int, bool, double, String and their list");
```

As sending **inputData** parameters only takes primitive types int,bool,string and array. But note that array also need primitive types like list of array of string. Then send data by calling function in like place order button.

**BackoffPolicy** In above code you have seen.It is the recurring event after first time the function has been called.After how much time it should be executed. You can also supply **constraints** like when to call function like when network is 
connected.

Then if you have gone through workmanger package it will guide about initializing the package. If not here is what 
initializtion on our code looks like.

```dart
import 'dart:convert';
import 'dart:developer';
import 'package:app/app.dart';
import 'package:app/common/common.dart';
import 'package:app/core/core.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:workmanager/workmanager.dart';

import 'ui/features/orders/bloc/order_manage_bloc.dart';

const mySimpleTask = "addWithData";
@pragma('vm:entry-point')
void callbackDispatcher() {
  Workmanager().executeTask((task, inputData) async {
    switch (task) {
      case mySimpleTask:
        try {
          
          await Firebase.initializeApp();
          await initApp();
          log("this method was called from native!");

          log("$mySimpleTask was executed. inputData = $inputData");

          log(inputData?['array']);
          final List<dynamic> cartItemsJson = await jsonDecode(inputData?['array']);

          final List<CartItem> cartItems = cartItemsJson.map((json) => CartItem.fromJson(json)).toList();

          await getIt<CartRepository>().placeOrder(cartItems, inputData?['string']);

          await getIt<CartRepository>().clear();

          return Future.value(true);
        } catch (e) {
          log("this method was called from native! with false");
          return Future.value(false);
        }
    }

    //Return true when the task executed successfully or not
    return Future.value(false);
  });
}



Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  await Firebase.initializeApp();
  await initApp();
  await Workmanager().initialize(
    callbackDispatcher,
    isInDebugMode: true,
  );
  runApp(
    BlocProvider(
      create: (context) => getIt<OrderManageBloc>(),
      child: const App(),
    ),
  );
}
```
In above code in **switch case statement** of **try block** you can see i am initalizing **firebase** and **init** function which contains injectors of the app.

#### Why i am initializing firebase and service locators or init function?
You can see i am calling api request which need authorization token and of course for that http request **dio** in my 
case needs to be initialized and also the cart items which i have saved offline as I said before are there. That is the 
reason I am calling them.


After that you can see there I am decoding the json that I have sent from **Place Order** and also I am returning 
**Future.value(true)** which indicates sucess and in other case i am returning it **false**.

**Note**: You may not get instant notification.In andorid you can see notification or register this type of task only after **15 mins** this is all controlled by android system whereas for your info in **ios** it is said to be **30 mins*
but you can debug through simultor in Xcode. I will write seprate tutorial for that

With this you can sucessfully call any task whether it is **periodic** or **oneOff**.
You can learn more about workmanager in its documentation. 

I am writing this blog because you can have overview of my work  and documentation contains basic examples which are enough but with this you can have quick idea about initializing app,dependencies and how we can pass other types of data rather than primitive types.

I will try to write it for ios soon.


