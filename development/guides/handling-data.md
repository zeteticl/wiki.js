---
title: Handling Data: Flags, Settings, and Files
description: A primer on the different ways to handle data in Foundry VTT.
published: true
date: 2021-06-17T15:02:08.857Z
tags: 
editor: markdown
dateCreated: 2021-03-25T15:03:13.490Z
---

This article is primarily geared towards Module development, but Systems can make use of its contents as well. It largely ignores the options for data storage that Systems have that come with being able to control Entity types via the system template.json.

## Data Storage Flowchart

```plantuml
@startuml

(*)-> "I have data I want to store"
if "It's a lot of data" then
	->[Yes] "JSON/FileUpload"
else
  if "Data is asociated with an Entity" then
    ->[Yes] "Flag"
  else
    -->[No] "Setting"
    if "Data needs to be shared between clients"
      ->[No] "Setting, scope 'client'"
    else
      if "Only GM can modify data"
        ->[Yes] "Setting, scope 'world'"
      else
        -->[No] "Setting, scope 'world' + GM proxy"
      endif
    endif
  endif
endif

@enduml
```

Use case flowchart:
1. I want to store a _lot_ of data and don't want it to affect performance -> **FileUpload with JSON**. Note that permissions might be set up on the world such that only the GM can upload files.
2. I want to store data associated with a particular entity -> **Flag**. Some entities are not editable by all clients and flags respect that.
3. I want to store data not associated with a particular entity:
	  1. The data does not have to be shared between clients -> **Setting, scope `client`**.
 	  2. The data does have to shared between clients:
		    1. All clients can access and modify -> **Setting with a GM Proxy** is the only way to do this, it is not pretty.
		    2. All clients can access but only the GM can modify -> **Setting, scope `world`**, no workarounds needed.


## Flags

Flags are the safest way that modules can store arbitrary data on existing entities. If you are making a module which allows the user to set a data point which isn't supported normally in Foundry Core or a system's data structure, you should use a flag.

A flag does not have to be a specific type, anything which can be `JSON.stringify`ed is valid.

> Lots of things in Foundry Core are entities and can thus accept flags, not just Actors, Items, and things Systems normally interact with. There is a list of Entity Subclasses on the [api page for Entity](https://foundryvtt.com/api/Entity.html).

### Setting a flag's value
Flags are automatically namespaced within the first parameter given to [`Entity#setFlag`](https://foundryvtt.com/api/Entity.html#setFlag).

```js
const newFlagValue = 'foo';

someEntity.setFlag('myModuleName', 'myFlagName', newFlagValue);
```

#### Can I mutate the value itself?
Setting a flag's value without `setFlag` will not persist that change in the database. This should only be done as part of a larger operation which persists an entity's data in the database, for example as a part of character sheet editing.

### Getting a flag's value
There are two places to get a flag value: On the data model itself, or with [`Entity#getFlag`](https://foundryvtt.com/api/Entity.html#getFlag)

```js
const flagValue = someEntity.getFlag('myModuleName', 'myFlagName');
// flagValue === 'foo'
```

> Beware that if you are attempting to get a flag for a module which is not installed/activated `getFlag` will `throw` rather than return `undefined`. If you need a flag from a module which might not exist, it is safer to look for the data on the date model itself.
{.is-danger}

#### On the data model itself
This can be somewhat tricky as it might be different depending on what entity you're dealing with. Somewhere in the entity's data object there is a `flags` key. The object attached is keyed by module `name`, which is itself an object keyed by flag name, as registered in `setFlag`.

### Unset a flag
A safe way to delete your flag's value is with [`Entity#unsetFlag`](https://foundryvtt.com/api/Entity.html#unsetFlag). This will fully delete that key from your module's flags on the provided entity.

```js
someEntity.unsetFlag('myModuleName', 'myFlagName');
```

### How do I use this?
It's arbitrary data that you can safely control on any `Entity`. Because of this, all of the hooks related to that entity are going to have your flag available when they fire.

For example, if I have a flag on a Scene, I can check if that flag exists when the `updateScene` hook fires.

```js
Hooks.on('updateScene', (scene, data) => {
  if (hasProperty(data, 'flags.myModule')) {
    console.log(data);
  }
});
```

## Settings

Settings, like flags, are a way for modules to store and persist data. Settings are not tied to a specific entity however, unlike flags. Also unlike flags they are able to leverage the 'scope' field to keep a set of data specific to a user's localStorage (`scope: client`) or put that data in the database (`scope: world`).

For the vast majority of use-cases, settings are intended to be modified by a UI, either a Menu or within the Module Settings panel itself. These settings are intended to be used to modify the functionality of a module or system, rather than store arbitrary data for that module or system.

### Registering a Setting

All settings must be registered before they can be set or accessed. This needs to be done with [`game.settings.register`](https://foundryvtt.com/api/ClientSettings.html#register), with `game.settings` being an instance of `ClientSettings`.

```js
/*
 * Create a custom config setting
 */
await game.settings.register('myModuleName', 'mySettingName', {
  name: 'My Setting',
  hint: 'A description of the registered setting and its behavior.',
  scope: 'world',     // "world" = sync to db, "client" = local storage 
  config: true,       // false if you dont want it to show in module config
  type: Number,       // Number, Boolean, String,  
  default: 0,
  range: {.           // range turns the UI input into a slider input
    min: 0,           // but does not validate the value
    max: 100,
    step: 10
  },
  onChange: value => { // value is the new value of the setting
    console.log(value)
  },
  filePicker: false,	// set true with a String `type` to use a file picker input
});
```

### Setting a Setting's value
Settings can be set with [`game.settings.set`](https://foundryvtt.com/api/ClientSettings.html#set). It's important to note that a `scope: world` setting can only be set by a Gamemaster user, and that `scope: client` settings will only persist on the user's local machine.

```js
const whateverValue = 'foo';

game.settings.set('myModuleName','myModuleSetting', whateverValue);
```

### Getting a Setting's value

Settings can be read with [`game.settings.get`](https://foundryvtt.com/api/ClientSettings.html#get). 

```js
const someVariable = game.settings.get('myModuleName','myModuleSetting');
console.log(someVariable); // expected to be 'foo'
```

### Reloading after setting a setting
Sometimes a setting's value changing means that the foundry client should reload for it to take effect. This can be accomplished with the `onChange` callback but should be debounced to allow for the case where several changed settings require a reload.

```js
const debouncedReload = foundry.utils.debounce(window.location.reload, 100);

game.settings.register("myModuleName", "myModuleSetting", {
  // ...,
  onChange: debouncedReload
}

game.settings.register("myModuleName", "myOtherSetting", {
  // ...,
  onChange: debouncedReload
}
```


### Setting Menus

Sometimes your use case is more complex than a few settings will allow you to manage. In these cases the best play is to register a settings menu with [`game.settings.registerMenu`](https://foundryvtt.com/api/ClientSettings.html#registerMenu), and manage your settings logic with a [FormApplication](https://foundryvtt.wiki/en/development/guides/understanding-form-applications). Note that we aren't actually registering a setting to be stored, simply a menu button.

This works best when used in conjunction with a registered setting of type `Object`.

```js
game.settings.registerMenu("myModule", "mySettingsMenu", {
  name: "My Settings Submenu",
  label: "Settings Menu Label",      // The text label used in the button
  hint: "A description of what will occur in the submenu dialog.",
  icon: "fas fa-bars",               // A Font Awesome icon used in the submenu button
  type: MySubmenuApplicationClass,   // A FormApplication subclass
  restricted: true                   // Restrict this submenu to gamemaster only?
});


await game.settings.register('myModuleName', 'myComplexSettingName', {
  scope: 'world',     // "world" = sync to db, "client" = local storage 
  config: false,      // we will use the menu above to edit this setting
  type: Object,
  default: {},        // can be used to set up the default structure
});


/**
 * For more information about FormApplications, see:
 * https://foundryvtt.wiki/en/development/guides/understanding-form-applications
 */
class MySubmenuApplicationClass extends FormApplication {
  // lots of other things...
  
  getData() {
  	return game.settings.get('myModuleName', 'myComplexSettingName');
  }
  
  _updateObject(event, formData) {
    const data = expandObject(formData);
    game.settings.set('myModuleName', 'myComplexSettingName', data);
  }
}
```

#### Why would I want this?

FormApplications allow you to run any logic you want, which includes setting settings, thus this kind of power could be leveraged to accomplish many things:

1. **Space.** You could easily use this to tidy up a lot of module settings which would otherwise take up a lot of vertical space on the settings list.
2. **Validation.** Since you control the FormApplication's submit logic, you could run validation on user inputs before saving them to the database.
3. **Edit Setting Objects.** If you have a use case for a complex object of data being stored as a setting, a FormApplication menu would let your users manipulate that object directly.


## JSON and FileUpload

A third option exists for modules and systems which deal with large amounts of very bulky data that is prefered so as not to clutter up the settings database with vast amounts of 'useless' data if the module is ever removed.

> Stub. Example can be found here: https://github.com/flamewave000/dragonflagon-fvtt/blob/master/df-chat-enhance/src/archive/DFChatArchive.ts
