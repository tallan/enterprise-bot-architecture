# Table of Contents

- [Learning the Basics](#the-basics)
- [Publishing the MVP](#publishing-the-minimum-viable-product)

# The Basics

This is a quick introduction to some of the vocabulary associated to Microsoft Bot Framework v4 and the components required to make a successful chatbot.  The goal of this is to see the parts in play for a basic echo bot, and then the components required to publish it.

A Bot Framework chatbot is simply a Web API with a single endpoint.  The endpoint is receives **Activities** from **Channels** that the bot can communicate with. 

------

> ###### KEY FOR UNDERSTANDING
>
> The **Activity** is the action that the bot will consume to take action.  Activities have many properties; the most important of which is arguably type.  The type property helps shape the behavior of the bot given its reception.  
>
> The **Channel** is the outlet that you can reach the bot through, examples of channels are webchat widgets, Facebook messenger, slack, skype, etc.
>
> The **Turn** is what Bot Framework v4 calls an incoming activity and the processing associated with creating a response.
>
> Combining these is what makes a bot work - **Activities** come from **channels** to the bot and are processed in **turns**.

------

When an activity is sent to a Bot it ultimately arrives at a hook called **OnTurnAsync**. Most templates follow some form of what is seen below.  Activities of type "Message" are handled with responses, all others are lumped together to be ignored, or in the case for learning, just signaled in the chat.

```c#
// This was taken from the Bot Builder V4 SDK Template Version 4.2.2 - Echo Bot
// so it may look different than the code in your template
public async Task OnTurnAsync(ITurnContext turnContext, 
                              CancellationToken cancellationToken)
{
    if (turnContext.Activity.Type == ActivityTypes.Message)
    {
        // Get the conversation state from the turn context.
        var state = await _accessors.CounterState.GetAsync(
            turnContext, () => new CounterState()
        );

        // Bump the turn count for this conversation.
        state.TurnCount++;

        // Set the property using the accessor.
        await _accessors.CounterState.SetAsync(turnContext, state);

        // Save the new turn count into the conversation state.
        await _accessors.ConversationState.SaveChangesAsync(turnContext);

        // Echo back to the user whatever they typed.
        var responseMessage = $"Turn {state.TurnCount}: You sent '{turnContext.Activity.Text}'\n";
            await turnContext.SendActivityAsync(responseMessage);
    }
    else
    {
        // possibly take some other action - maybe system related
        await turnContext.SendActivityAsync(
            $"{turnContext.Activity.Type} event detected"
        );
    }    
}
```

In the sample above, the bot simply has a counter, which is incremented whenever a message comes to the bot, and then the bot echoes the message back to the user with the count.  If any other activity type comes into the bot, the bot simply responds with the message that the event type was detected.

![Preview](https://imagesforgithub.blob.core.windows.net/images/basicConvo.png)

Something noteworthy to mention in the photo above.  The first thing that happened when I opened the chat was a "conversationUpdate" event, then I said "hi," and then there was another.  These two events are actually myself and the bot joining the conversation, which will always be the case.  This is also an easy indicator of there being something wrong in the codebase with the bot, since this activity should happen before any message exchange occurs. (It is worth mentioning that production worthy bots will very likely not have these messages, they are just great for during dev)

------

The only thing left out of this equation for understanding is **State**. In the code snippet above, you can see there is a State object which maintains the count value.  There are three different type of states which your bot can manage.  

- **User State** - available in any turn for a user in a specific channel regardless of conversation
- **Conversation State** - available in any turn for any user in a specific channel regardless of user 
- **Private Conversation State** - available for a user within a specific channel scoped to a single conversation

Keep in mind that it is important to think about what information you want maintained by the bot, some items make sense in user state such as a user's name, others such as what kind of sandwich the user wants to purchase, may only make sense in the scope of a single conversation.

------

> ###### KEY FOR UNDERSTANDING 
>
> Users are defined by the interaction through a given channel, thus technically the same user communicating through two different channels, will be perceived as two separate users to the bot, which means that **State will not be shared between channels**.
>
> ------

------

# Publishing the Minimum Viable Product

Now that you understand the basics pieces of a bot, let's look at what is actually required to make a bot available to the public.  Keeping in line with the title, we'll stick with deploying the unmodified template, or as close to it as we can.

Likely the only thing that will actually be required to modify in the template will be to do with the storage of state.  Generally bot templates set the storage location to be in memory, which works great for dev, as there is no need to provision resources or do any type of configuration; just run the bot and it works.  Deploying a bot requires a real data store for state.

------

> ###### KEY FOR UNDERSTANDING
>
> Storage is what maintains more than just things like the username or another detail pushed into the state.  Behind the scenes the state also holds onto the stack maintaining the conversation itself.  What this means is that if storage is not in place and the conversation will always start from the root, and never go further, since the bot will not remember the previous transaction occurring.

------

There are any number of ways to configure the storage for a bot, but the easiest in my opinion is with Azure Blob Storage, especially since the template likely holds commented code for leveraging it.  See the code below.

In Startup.cs, unmodified.  See all the comments and that they refer to blob storage configuration.

```c#
// Memory Storage is for local bot debugging only. When the bot
// is restarted, everything stored in memory will be gone.
IStorage dataStore = new MemoryStorage();

// For production bots use the Azure Blob or
// Azure CosmosDB storage providers. For the Azure
// based storage providers, add the Microsoft.Bot.Builder.Azure
// Nuget package to your solution. That package is found at:
// https://www.nuget.org/packages/Microsoft.Bot.Builder.Azure/
// Un-comment the following lines to use Azure Blob Storage
// // Storage configuration name or ID from the .bot file.
// const string StorageConfigurationId = "<STORAGE-NAME-OR-ID-FROM-BOT-FILE>";
// var blobConfig = botConfig.FindServiceByNameOrId(StorageConfigurationId);
// if (!(blobConfig is BlobStorageService blobStorageConfig))
// {
//    throw new InvalidOperationException($"The .bot file does not contain an blobstorage with name '{StorageConfigurationId}'.");
// }
// // Default container name.
// const string DefaultBotContainer = "<DEFAULT-CONTAINER>";
// var storageContainer = string.IsNullOrWhiteSpace(blobStorageConfig.Container) ? DefaultBotContainer : blobStorageConfig.Container;
// IStorage dataStore = new Microsoft.Bot.Builder.Azure.AzureBlobStorage(blobStorageConfig.ConnectionString, storageContainer);

// Create and add conversation state.
var conversationState = new ConversationState(dataStore);
services.AddSingleton(conversationState);
```

To take advantage of this, the first step is to provision an Azure Storage account.

[Link to Create Azure Storage](https://portal.azure.com/#create/Microsoft.StorageAccount-ARM)

Once the account is created, this code can be changed, and a change will need to be made to the .bot file.

```json
{
  "name": "EchoBot",
  "services": [
    {
      "type": "endpoint",
      "name": "development",
      "endpoint": "http://localhost:3978/api/messages",
      "appId": "",
      "appPassword": "",
      "id": "1"
    },
    {
      "COMMENT": "The following section is needed to identify the blob storage"
    },
    {
      "type": "blob",
      "name": "SOME NAME - NEEDS TO MATCH THE CODE IN STARTUP BELOW",
      "connectionString": "YOUR CONNECTION STRING - GET FROM KEYS IN THE AZURE PORTAL",
      "container": "THE TARGET CONTAINER NAME"
    },
    {
      "COMMENT": "----------END OF NEW SECTION----------"
    }
  ],
  "padlock": "",
  "version": "2.0"
}
```

This is the way the code will be modified in Startup.cs.

```c#
// Memory Storage is for local bot debugging only. When the bot
// is restarted, everything stored in memory will be gone.
//IStorage dataStore = new MemoryStorage();

// For production bots use the Azure Blob or
// Azure CosmosDB storage providers. For the Azure
// based storage providers, add the Microsoft.Bot.Builder.Azure
// Nuget package to your solution. That package is found at:
// https://www.nuget.org/packages/Microsoft.Bot.Builder.Azure/
// Un-comment the following lines to use Azure Blob Storage
// Storage configuration name or ID from the .bot file.
const string StorageConfigurationId = "SOME NAME TO MATCH THE .bot FILE";
var blobConfig = botConfig.FindServiceByNameOrId(StorageConfigurationId);
if (!(blobConfig is BlobStorageService blobStorageConfig))
{
    throw new InvalidOperationException($"The .bot file does not contain an blob storage with name '{StorageConfigurationId}'.");
}
// Default container name.
const string DefaultBotContainer = "A Container Name in YOUR STORAGE";
var storageContainer = string.IsNullOrWhiteSpace(blobStorageConfig.Container) ? DefaultBotContainer : blobStorageConfig.Container;
IStorage dataStore = new Microsoft.Bot.Builder.Azure.AzureBlobStorage(blobStorageConfig.ConnectionString, storageContainer);

// Create and add conversation state.
var conversationState = new ConversationState(dataStore);
services.AddSingleton(conversationState);
```

Now that storage is hooked up, the application can be deployed to Azure.  It is a simple app, so in Visual Studio, right-click publish to an App Service will be all that is needed.  Now that the app is published, it could be communicated with given a platform that can connect to it.  That being said, what would be used to connect... an **Azure Bot Channels Registration**!  

[Link to Create Bot Channels Registration](https://portal.azure.com/#create/Microsoft.BotServiceConnectivityGalleryPackage)

------

> ###### KEY FOR UNDERSTANDING
>
> The Azure Bot Channels Registration serves more than one purpose.  First and foremost it becomes a test bench and gateway for connecting to a bot.  The channels registration also creates a client secret and app Id that get added to the .bot file, and what this does is create a layer of authentication to your app, preventing unregistered entities from connecting.  Channels now get connected to the bot through the channel registration, which also works as a universal translator.  The bot consumes activities and each channel uses their own version of those activities, so the channels registration both translates Bot Framework's activities to the channel's activity, but then also back when the activity comes from the channel.
>
> ------

The channels registration now expects that app Id and app password to be used by the bot.  Therefore the .bot file needs to be edited and then republished with the correct values (See below).

```json
{
  "name": "EchoBot",
  "services": [
    {
      "type": "endpoint",
      "name": "development",
      "endpoint": "http://localhost:3978/api/messages",
      "appId": "REQUIRED --------> THE APP ID",
      "appPassword": "REQUIRED --> THE APP PASSWORD",
      "id": "1"
    },
    {
      "COMMENT": "The following section is needed to identify the blob storage"
    },
    {
      "type": "blob",
      "name": "SOME NAME - NEEDS TO MATCH THE CODE IN STARTUP BELOW",
      "connectionString": "YOUR CONNECTION STRING - GET FROM KEYS IN THE AZURE PORTAL",
      "container": "THE TARGET CONTAINER NAME"
    },
    {
      "COMMENT": "----------END OF NEW SECTION----------"
    }
  ],
  "padlock": "",
  "version": "2.0"
}
```

Once the republish is complete, the connection can be validated in the channels registration within the Azure Portal.  The channels registration has a tab named "Test in Web Chat" which allows you to communicate with the bot.

![testinwebchat](https://imagesforgithub.blob.core.windows.net/images/testinwebchat.png)

The bot is now publicly available. Underneath the tab channels there are instructions for connecting different platforms to the bot, such as Facebook, Skype, Email, etc, all of which have unique configurations binding the channel to the channels registration, thus creating a secure line of communication.

------

# Taking a Look at the Big Picture

Now that the bare minimum is understood, the real work can begin.  Microsoft released this image of their proposed architecture, and it will be followed as a guide with real work and examples of how to best go about it.

![bot architecture diagram](https://imagesforgithub.blob.core.windows.net/images/conversational-bot.png)

