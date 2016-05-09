# Overview

Synchronized arrays should be used for any list of objects that will be sorted, iterated, and which have unique IDs. The synchronized array assumes that items are added using [`$add()`](https://www.firebase.com/docs/web/libraries/angular/api.html#angularfire-firebasearray-addnewdata), and that they will therefore be keyed using Firebase [push IDs](https://www.firebase.com/docs/web/guide/saving-data.html#section-push).

We create a synchronized array with the `$firebaseArray` service. The array is [sorted in the same order](https://www.firebase.com/docs/web/guide/retrieving-data.html#section-ordered-data) as the records on the server. In other words, we can pass a [query](https://www.firebase.com/docs/web/guide/retrieving-data.html#section-queries) into the synchronized array, and the records will be sorted according to query criteria.

While the array isn't technically read-only, it has some special requirements for modifying the structure (removing and adding items) which we will cover below. Please read through this entire section before trying any slicing or dicing of the array.

```js
// define our app and dependencies (remember to include firebase!)
var app = angular.module("sampleApp", ["firebase"]);
// inject $firebaseArray into our controller
app.controller("ProfileCtrl", ["$scope", "$firebaseArray",
  function($scope, $firebaseArray) {
    var messagesRef = new Firebase("https://<YOUR-FIREBASE-APP>.firebaseio.com/messages");
    // download the data from a Firebase reference into a (pseudo read-only) array
    // all server changes are applied in realtime
    $scope.messages = $firebaseArray(messagesRef);
    // create a query for the most recent 25 messages on the server
    var query = messagesRef.orderByChild("timestamp").limitToLast(25);
    // the $firebaseArray service properly handles database queries as well
    $scope.filteredMessages = $firebaseArray(query);
  }
]);
```

We can now utilize this array as expected with Angular directives.

```html
<ul>
  <li ng-repeat="message in messages">{{ message.user }}: {{ message.text }}</li>
</ul>
```

To add a button for removing messages, we can make use of `$remove()`, passing it the message we want to remove:

```html
<ul>
  <li ng-repeat="message in messages">
    {{ message.user }}: {{ message.text }}
    <button ng-click="messages.$remove(message)">x</button>
  </li>
</ul>
```

We also have access to the key for the node where each message is located via `$id`:

```html
<ul>
  <li ng-repeat="message in messages">
    Message data located at node /messages/{{ message.$id }}
  </li>
</ul>
```

# API Summary

The table below highlights some of the common methods on the synchronized array. The complete list of methods can be found in the [API documentation](https://www.firebase.com/docs/web/libraries/angular/api.html#angularfire-firebasearray) for `$firebaseArray`.

| Method  | Description |
| ------------- | ------------- |
| [$add(data)](https://www.firebase.com/docs/web/libraries/angular/api.html#angularfire-firebasearray-addnewdata) | Creates a new record in the array. Should be used in place of `push()` or `splice()`. |
| [$remove(recordOrIndex)](https://www.firebase.com/docs/web/libraries/angular/api.html#angularfire-firebasearray-removerecordorindex) | Removes an existing item from the array. Should be used in place of `pop()` or `splice()`. |
| [$save(recordOrIndex)](https://www.firebase.com/docs/web/libraries/angular/api.html#angularfire-firebasearray-saverecordorindex) | Saves an existing item in the array. |
| [$getRecord(key)](https://www.firebase.com/docs/web/libraries/angular/api.html#angularfire-firebasearray-getrecordkey) | Given a Firebase database key, returns the corresponding item from the array. It is also possible to find the index with `$indexFor(key)`. |
| [$loaded()](https://www.firebase.com/docs/web/libraries/angular/api.html#angularfire-firebasearray-loaded) | Returns a promise which resolves after the initial records have been downloaded from our database. This is only called once and should be used with care. See [Extending the Services](extending-services.md) for more ways to hook into server events. |

# Meta Fields on the Object

Similar to synchronized objects, each item in a synchronized array will contain the following special attributes:

| Method  | Description |
| ------------- | ------------- |
| $id | The key for each record. This is equivalent to each record's path in our database as it would be returned by `ref.key()`. |
| $priority | The [priority](https://www.firebase.com/docs/web/guide/retrieving-data.html#section-ordered-data) of each child node is stored here for reference. Changing this value and then calling `$save()` on the record will also change the priority on the server and potentially move the record in the array. |
| $value | If the data for this child node is a primitive (number, string, or boolean), then the record itself will still be an object. The primitive value will be stored under `$value` and can be changed and saved like any other field. |

# Modifying the Synchronized Array

The contents of this array are synchronized with a remote server, and AngularFire handles adding, removing, and ordering the elements. Because of this special arrangement, AngularFire provides the concurrency safe `$add()`, `$remove()`, and `$save()` methods to modify the array and its elements.

Using methods like `splice()`, `pop()`, `push()`, `shift()`, and `unshift()` will probably work for modifying the local content, but those methods are not monitored by AngularFire and changes introduced won't affect the content or order on the remote server. Therefore, to change the remote data, the concurrency-safe methods should be used instead.

```js
var messages = $FirebaseArray(ref);
// add a new record to the list
messages.$add({
  user: "physicsmarie",
  text: "Hello world"
});
// remove an item from the list
messages.$remove(someRecordKey);
// change a message and save it
var item = messages.$getRecord(someRecordKey);
item.user = "alanisawesome";
messages.$save(item).then(function() {
  // data has been saved to our database
});
```

# Full Example

Using those methods together, we can synchronize collections between multiple clients, and manipulate the records in the collection:

```js
var app = angular.module("sampleApp", ["firebase"]);

app.factory("chatMessages", ["$firebaseArray",
  function($firebaseArray) {
    // create a reference to the database where we will store our data
    var randomRoomId = Math.round(Math.random() * 100000000);
    var ref = new Firebase("https://docs-sandbox.firebaseio.com/af/array/full/" + randomRoomId);

    return $firebaseArray(ref);
  }
]);

app.controller("ChatCtrl", ["$scope", "chatMessages",
  function($scope, chatMessages) {
    $scope.user = "Guest " + Math.round(Math.random() * 100);

    $scope.messages = chatMessages;

    $scope.addMessage = function() {
      // $add on a synchronized array is like Array.push() except it saves to the database!
      $scope.messages.$add({
        from: $scope.user,
        content: $scope.message,
        timestamp: Firebase.ServerValue.TIMESTAMP
      });

      $scope.message = "";
    };

    // if the messages are empty, add something for fun!
    $scope.messages.$loaded(function() {
      if ($scope.messages.length === 0) {
        $scope.messages.$add({
          from: "Uri",
          content: "Hello!",
          timestamp: Firebase.ServerValue.TIMESTAMP
        });
      }
    });
  }
]);
```

```html
<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.3.15/angular.min.js"></script>
<script src="https://cdn.firebase.com/js/client/2.2.4/firebase.js"></script>
<script src="https://cdn.firebase.com/libs/angularfire/1.2.0/angularfire.min.js"></script>

<div ng-app="sampleApp" ng-controller="ChatCtrl">
  <p>
    Sort by:
    <select ng-model="orderBy">
      <option>from</option>
      <option>content</option>
      <option>timestamp</option>
    </select>
  </p>

  <p>Search: <input ng-model="searchText"></p>

  <ul class="chatbox">
    <!-- arrays are fully sortable and filterable -->
    <li ng-repeat="message in messages | orderBy:orderBy | filter:searchText">
      {{ message.from }}: {{ message.content }}

      <!-- delete a message using $remove -->
      <a href="" ng-click="messages.$remove(message)">X</a>
    </li>
  </ul>

  <form ng-submit="addMessage()">
    <input type="text" ng-model="message">
    <input type="submit">
  </form>
</div>
```

Head on over to the [API reference](https://www.firebase.com/docs/web/libraries/angular/api.html#angularfire-firebasearray) for `$firebaseArray` to see more details for each API method provided by the service. Now that we have a grasp of synchronizing data with AngularFire, the [next section](user-auth.md) of this guide moves on to a different aspect of building applications: user authentication.