# Updates and Followups

According to the [developer portal](https://discord.com/developers/docs/interactions/receiving-and-responding#responding-to-an-interaction), our bot cannot take too long to respond to an interaction. More specifically, no longer than 3 seconds. To confirm this, I made my program intentionally halt for three seconds by adding the following line (using the same code as from the previous page).

```rs
use std::{thread, time::Duration};

// ...
            if cmd.name == "ping" {
                thread::sleep(Duration::from_secs(3));

                // ...
            }
// ...
```

Sure enough, once I booted up the bot and called `/ping` in my test server, I get the "The application did not respond error", but more importantly, our program crashed. This is because of this line:

```rs
// ...
            if cmd.name == "ping" {
                // ...

                interaction_client.create_response(id, token, &response).await?;
            }
// ...
```

By the time the three seconds passed, Discord "forgot" the original interaction, so it doesn't exist anymore and cannot be replied. Thus, once we await for the response result, we get an error, and because of the `?`, we return the main function with the following error displayed to the console.

```
Error: Response error: status code 404, error: {"message": "Unknown interaction", "code": 10062}
```

Thus, if our bot does something very clever that takes a bit of time, it's always good to ensure we respond with at least something, such as a "Loading!". From what I've found, it seems it can be done with the [`DEFERRED_CHANNEL_MESSAGE_WITH_SOURCE`](https://docs.rs/twilight-model/0.15.4/twilight_model/http/interaction/enum.InteractionResponseType.html#variant.DeferredChannelMessageWithSource) response variant.

Naturally, after we give that response, we need to update it to actually include the response of the command. Again, from what I've found, this is done with [`update_response`](https://docs.rs/twilight-http/0.15.4/twilight_http/client/struct.InteractionClient.html#method.update_response), which only needs the interaction token, so I'm assuming the Discord API can identify the exact message that needs to be updated? We'll find out.

For this test, we'll make a command called `hello` that responds with a cute gif of a yellow pegasus waving back. Purely for testing, we'll still sleep for a long time to really ensure everything works as intended (and our interaction won't get yeeted out of existence). To ensure we're on the same page, this is what we'll start with.

```rs
use std::{env, thread, time::Duration};
use twilight_http::Client;
use twilight_gateway::{Event, Intents, Shard, ShardId};
use twilight_model::{application::interaction::InteractionData, http::interaction::*};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // Initialize the tracing subscriber.
    tracing_subscriber::fmt::init();

    tracing::info!("started shard");

    let client = Client::new(env::var("DISCORD_TOKEN")?);

    let application_id = {
        let response = client.current_user_application().await?;
        response.model().await?.id
    };
    
    let interaction_client = client.interaction(application_id);

    interaction_client.create_global_command()
        .chat_input("hello", "Say hello to KikiBot!")?
        .await?;

    let intents = Intents::GUILD_MESSAGES;
    let mut shard = Shard::new(ShardId::ONE, env::var("DISCORD_TOKEN")?, intents);

    loop {
        let event = match shard.next_event().await {
            Ok(event) => event,
            Err(source) => {
                tracing::warn!(?source, "error receiving event");

                if source.is_fatal() {
                    break;
                }

                continue;
            },
        };

        if let Event::InteractionCreate(ref interaction) = event {
            let id = interaction.id;
            let token = &interaction.token;
            if let Some(ref interaction_data) = interaction.data {
                if let InteractionData::ApplicationCommand(ref cmd) = interaction_data {
                    match cmd.name.as_str() {
                        "ping" => {
                            let response = InteractionResponse {
                                kind: InteractionResponseType::ChannelMessageWithSource,
                                data: Some(InteractionResponseData {
                                    content: Some("Pong!!!! :horse:".to_string()),
                                    ..Default::default()
                                })
                            };
            
                            interaction_client.create_response(id, token, &response).await?;
                        },
                        "hello" => {
                            // our code goes here
                        },
                        _ => {}
                    }
                }
            }
        } else {
            tracing::info!(?event, "received unknown event");
        }
    }

    Ok(())
}
```

Notice we start with the same code as usual, then call `create_global_command` on the interaction client to create the `hello` command. Remember, we need to ensure this code doesn't run again after the command is created, to avoid sending repetitive command creation requests to the Discord API. Other than that, I changed the actual handling of the command into a `match` statement, since we now have two commands and I absolutely despise `else if`. Our task is to add things to the `// our code goes here` part.

First things first, we create the deferred message response right away. Then, we add an artificial sleep of say, 10 seconds. Lastly, we need to update the same response with the corresponding gif we wish to send. Unlike `create_response`, where we have to give the response inside the method itself, the `update_response` method actually gives us an [`UpdateResponse`](https://docs.rs/twilight-http/0.15.4/twilight_http/request/application/interaction/struct.UpdateResponse.html) struct which is a builder-type struct. We use that to construct the updated message, and since all we're updating is the message's content (from nothing to the gif's URL), we just include the URL in the content and send it. It all looks like this.

```rs
        "hello" => {
            let initial_response = InteractionResponse {
                kind: InteractionResponseType::DeferredChannelMessageWithSource,
                // deferred messages don't require any data
                data: None
            };

            interaction_client
                .create_response(id, token, &initial_response)
                .await?;

            // we artificially sleep for 10 seconds
            thread::sleep(Duration::from_secs(10));

            let gif = "https://cdn.discordapp.com/emojis/645667403291820044.gif?size=96&quality=lossless";

            interaction_client
                .update_response(token)
                .content(Some(gif))?
                .await?;
        },
```

Once my bot was up and running, I noticed the command didn't get added. Upon inspecting online, it seems I had to "reinvite" the bot with the same invite link again. Not sure if I'll have to do this everytime a new slash command is added but it indeed fix the issue. There's no need to kick or remove the bot from the server, just open the link again to reinvite the bot as if it wasn't there already.

Upon doing that, the "hello" command magically appeared without having to stop the program that was already running.

<video width="468" height="264" autoplay loop muted>
    <source src="./vid/hellobot.mp4" type="video/mp4">
</video>