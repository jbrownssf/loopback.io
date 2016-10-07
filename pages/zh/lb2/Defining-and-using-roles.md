---
title: "Defining and using roles"
lang: zh
layout: page
keywords: LoopBack
tags:
sidebar: zh_lb2_sidebar
permalink: /doc/zh/lb2/Defining-and-using-roles.html
summary:
---

LoopBack enables you to define both static and dynamic roles. Static roles are stored in a data source and are mapped to users. In contrast, dynamic roles aren’t assigned to users and are determined during access.

## Static roles

Here is an [example](https://github.com/strongloop/loopback-example-access-control/blob/master/server/boot/create-model-instances.js) defining a new static role and assigning a user to that role.

**/server/boot/script.js**

```js
User.create([{
  username: 'John',
  email: 'john@doe.com',
  password: 'opensesame'
}, {
  username: 'Jane',
  email: 'jane@doe.com',
  password: 'opensesame'
}, {
  username: 'Bob',
  email: 'bob@projects.com',
  password: 'opensesame'
}], function(err, users) {
  if (err) return cb(err);

  //create the admin role
  Role.create({
    name: 'admin'
  }, function(err, role) {
    if (err) cb(err);

    //make bob an admin
    role.principals.create({
      principalType: RoleMapping.USER,
      principalId: users[2].id
    }, function(err, principal) {
      cb(err);
    });
  });
});
```

Now you can use the role defined above in the access controls. For example, add the following to common/models/project.json to enable users in the "admin" role to call all REST APIs.

**/common/models/model.json**

```js
{
  "accessType": "EXECUTE",
  "principalType": "ROLE",
  "principalId": "admin",
  "permission": "ALLOW",
  "property": "find"
}
```

## Dynamic roles

Sometimes static roles aren’t flexible enough.  LoopBack also enables you to define _dynamic roles_ that are defined at run-time.

LoopBack provides the following built-in dynamic roles.

<table>
  <tbody>
    <tr>
      <th>Role object property</th>
      <th>String value</th>
      <th>Description</th>
    </tr>
    <tr>
      <td>Role.OWNER</td>
      <td>$owner</td>
      <td>Owner of the object</td>
    </tr>
    <tr>
      <td>Role.AUTHENTICATED</td>
      <td>$authenticated</td>
      <td>authenticated user</td>
    </tr>
    <tr>
      <td>Role.UNAUTHENTICATED</td>
      <td>$unauthenticated</td>
      <td>Unauthenticated user</td>
    </tr>
    <tr>
      <td>Role.EVERYONE</td>
      <td>$everyone</td>
      <td>Everyone</td>
    </tr>
  </tbody>
</table>

The first example used the “$owner” dynamic role to allow access to the owner of the requested project model. 

{% include note.html content="

To qualify a $owner, the target model needs to have a belongsTo relation to the owning model and property matching the foreign key of the target model instance. 

" %}

Use [`Role.registerResolver()`](http://apidocs.strongloop.com/loopback/#role-registerresolver) to set up a custom role handler in a [boot script](/doc/{{page.lang}}/lb2/6095038.html).  This function takes two parameters: 

1.  String name of the role in question.
2.  Function that determines if a principal is in the specified role. The function signature must be `function(role, context, callback)`.

For example, here is the role resolver from [loopback-example-access-control](https://github.com/strongloop/loopback-example-access-control/):

**/server/boot/script.js**

```js
module.exports = function(app) {
  var Role = app.models.Role;
  Role.registerResolver('teamMember', function(role, context, cb) {
    function reject(err) {
      if (err) {
        return cb(err);
      }
      cb(null, false);
    }
    if (context.modelName !== 'project') {
      // the target model is not project
      return reject();
    }
    var userId = context.accessToken.userId;
    if (!userId) {
      return reject(); // do not allow anonymous users
    }
    // check if userId is in team table for the given project id
    context.model.findById(context.modelId, function(err, project) {
      if (err || !project) {
        reject(err);
      }
      var Team = app.models.Team;
      Team.count({
        ownerId: project.ownerId,
        memberId: userId
      }, function(err, count) {
        if (err) {
          return reject(err);
        }
        cb(null, count > 0); // true = is a team member
      });
    });
  });
};
```

Using the dynamic role defined above, we can restrict access of project information to users that are team members of the project.

**/common/models/model.json**

```js
{
  "accessType": "READ",
  "principalType": "ROLE",
  "principalId": "teamMember",
  "permission": "ALLOW",
  "property": "findById"
}
```

<div class="sl-hidden"><strong>REVIEW COMMENT from Rand</strong><br>
  <p>I assume that you could define any number of role resolvers in a single boot script. Is that true?</p>
  <p>Need some more explanation:</p>
  <ul>
    <li>What is <code>context</code> and where does it come from?
      <ul>
        <li>is an object provided by loopback to give the user context into the request (ie. when the request comes in, they can access context.req or context.res, like you normally would with middleware)</li>
      </ul>
    </li>
    <li>What is purpose of process.nextTick() ?
      <ul>
        <li>this is part of node and requires a whole explanation into the heart of node, (ie the event loop). basically this delays the `cb` call until the next `tick` of the event loop, so the machinery can process all events currently in the queue before
          processing your callback.</li>
      </ul>
    </li>
    <li>what is return <code>reject()</code> and where does <code>reject()</code> come from?
      <ul>
        <li>reject is a function we define inline ie function reject() ...</li>
        <li>we basically say, if request is not a a request to api/projects (by checking the modelname), do not execute the rest of the script and reject the request (do nothing).</li>
        <li>do this by calling "reject", which will return false during the next cycle of the event loop (returning false in the second param means the person is NOT a team member, ie is not in that role)</li>
      </ul>
    </li>
    <li>The logic at the end that determines whether teamMember is in team based on Team.count() seems a bit convoluted. Explain, esp. how cb works in this specific case.
      <ul>
        <li>This example is provided verbatim by raymond</li>
        <li>but the idea is you have a team table and you do a query to count the "rows" because the requester can be on multiple teams, so any number you get greater than 0 is ok</li>
      </ul>
    </li>
  </ul>
</div>