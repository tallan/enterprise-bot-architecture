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
        var responseMessage = $"Turn {state.TurnCount}: You sent 
            '{turnContext.Activity.Text}'\n";
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

