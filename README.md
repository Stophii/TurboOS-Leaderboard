# TurboOS Leaderboard 

## Description

Set up a backend function to store scores and ids and a frontend function to display them as a leaderboard!

>💡 **Tip** 💡 You can reach the TurboOS website [here](https://os.turbo.computer/)!

## How it works

Turbo OS has two main features that we will be using to make a leaderboard. 

`os::server` - is used whenever you are trying to communicate with the Server.

`os::client` - is used whenever a client side action is required.

In most uses a `client` is going to call upon a function through the backend which will utilize a `server`.

In order for a function to have readable data it needs to take the data provided to it by the `client` and serialize it through `borsh` and then store it in the `server` as a `file`

The act of serializing data looks like this:

```rust
let score = state.score;
let id = state.name;
let data = MyStruct {
  score,
  id,
};

let bytes = borsh::to_vec(&data).unwrap();

```
and that filepath might look something like this

```rust
    let file_path = "file";
    let mut file = os::server::read_or!(Vec<MyStruct>, &file_path, Vec::new());
```
The `read_or!` macro will do one of two things: 

If there is a file already at the file_path, it will return that data, deserialized to the type that you include in the first paramater. In this case, that would be `Vec<MyStruct>.
If there is no file at that file_path, it will create a new file with the 3rd paramater as the data in the file.

If this is all a little over your head don't worry, it will make more sense once you have started to practice using it!

## Frontend Variables

The frontend of our project will include setting up variables for us to serialize into data the server can store. We will also need to execute a backend function from the frontend.

lets take it one step at a time. I want to store a score and an ID so I'll set up a struct to do just that:

```rust
#[derive(BorshDeserialize, BorshSerialize, Debug, Clone, PartialEq)]
pub struct ScoreData {
    pub score: u32,
    pub id: String,
}
```


In my `gamestate` I have a `score` and `id` field so when I need to define the values of this struct I'll use those variables!

I'll need to define those values when I want to submit a score. Even though we haven't setup the backend function yet I'll setup the call for it next.

```rust

    if gamepad(0).a.just_pressed() {
        let data = ScoreData {
            score: state.score,
            id: state.id.name.clone(),
        };

        let bytes = borsh::to_vec(&data).unwrap();
        os::client::exec(PROGRAM_ID, "submit_score", &bytes);

        state.score = 0;
    }
```

Calling the `submit_score` function is going to store a `score` and `id` in our server.

Lets write up the backend next to make this work!

## Backend Leaderboard

On the back end of our project we can write a simple `submit_score` function like this.

```rust
//backend.rs

use crate::*;

#[unsafe(export_name = "turbo/submit_score")]
unsafe extern "C" fn submit_score() -> usize {
    let file_path = "leaderboard";
    let mut leaderboard = os::server::read_or!(Vec<ScoreData>, &file_path, Vec::new());

    let score_data = os::server::command!(ScoreData);

    leaderboard.push(score_data.clone());
    leaderboard.sort_by(|a, b| b.score.cmp(&a.score));
    leaderboard.truncate(100);

    let Ok(_) = os::server::write!(&file_path, leaderboard) else {
        return os::server::CANCEL;
    };

    for entry in &leaderboard {
        os::server::log!("{}: {}", entry.id, entry.score);
    }

    return os::server::COMMIT;
}
```

This function records a `score` and `id` and writes it to a file we have specified as `leaderboard`.

In that file we can see 100 scores and ID's and we log it in the server using the `server::log!` command.

Thanks to our `ScoreData` struct we can serialize this data to be stored in the server.

```rust
pub struct ScoreData {
    pub score: u32,
    pub id: String,
}
```

To be clear, this will not display the scores on screen, it will only store them in a file on the server. 

We will need a seperate function on the client side to `watch` that file so we can display the data!

>💡 **Tip** 💡 You can view the log in the `live activity feed` in `your programs` in the [dashboard](https://os.turbo.computer/dashboard)!

## Watching Files

If you took a look inside your dashboard and saw your live activity feed you might see something like this.

<img width="1152" alt="Screenshot 2025-05-31 at 4 41 48 PM" src="https://github.com/user-attachments/assets/17aa2f5a-abc0-4473-b172-8b0a69ee8eaf" />

Make sure that it has the green border around it, because that means everything is working correctly!

Inside the dropdown of that function you might see:

```
Read file leaderboards/leaderboard (173 bytes).
Wrote file leaderboards/leaderboard (186 bytes).
Mantiray: 29002
Bat: 15363
Mantiray: 849
leaderboards::submit_score exited with 0 after 0.86ms
```

which is good, it means our `server::log` is working! Let's work on displaying this file to our game!

>💡 **Tip** 💡 I've assigned random names to my ID's which is why you see `Mantiray` and `Bat` for my log. `server::log` is extremely useful for debugging, so you should take advantage of it.

Now lets setup a `leaderboard` function!

```rust
pub fn leaderboard(state: &GameState) {
    text_box!(
        "Leaderboard",
        font = "medium",
        scale = 1.0,
        start = 0,
        end = 100,
        width = 200,
        height = 140,
        x = 100,
        y = 0,
        align = "center",
    );

    let leaderboard = os::client::watch_file(PROGRAM_ID, "leaderboard")
        .data
        .and_then(|file| <Vec<ScoreData>>::try_from_slice(&file.contents).ok())
        .unwrap_or_default();

    for (i, entry) in leaderboard.iter().take(10).enumerate() {
        let text = format!("{}. {} : {}", i + 1, entry.id, entry.score);
        text_box!(
            &text,
            font = "medium",
            scale = 1.0,
            start = 0,
            end = 100,
            width = 200,
            height = 140,
            x = 100,
            y = 10 + (i as i32 * 10),
            align = "center",
        );
    }
}
```

>💡 **Tip** 💡 Through use of the `os::client::watch_file` we can access (and then display) the data from the server.

Now in order to have this `leaderboard` display I can just call it in the `go::loop`

```rust
turbo::go! {
    let mut state = GameState::load();

    state.score += 1;

    let name = format!("ID:{}\n\nScore:{}", state.id.name, state.score);
    text!(&name, x = 10, y = 10, color = 0xffffffff);

    if gamepad(0).a.just_pressed() {
        let data = ScoreData {
            score: state.score,
            id: state.id.name.clone(),
        };

        let bytes = borsh::to_vec(&data).unwrap();
        os::client::exec(PROGRAM_ID, "submit_score", &bytes);

        state.score = 0;
    }

    leaderboard(&state); //<-- Leaderboard is being run here!

    state.save();
}
```

In my project the data displays like this.

<img width="880" alt="Screenshot 2025-05-31 at 4 58 18 PM" src="https://github.com/user-attachments/assets/a43b585d-6335-4c55-8fde-622c774ffb74" />

## Where to go from here

Now we have a project with TurboOS storing, reading, and displaying scores! All that's left is for you to come up with the game side of it.

Is it a procedurely generated dungeon crawler like [Dungeon Crashing](https://bingus-productions.itch.io/dungeon-crashing)?

Whatever you decide to come up with know that TurboOS is a powerful tool and you've but scratched the surface of backend integrated multiplayer!
