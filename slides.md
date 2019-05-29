---
class: center, middle

I don't think we should show the pieces diagrams again
Instead just enumerate them

also make a graphic showing aggregates 1 per id

many games, many teams, many checks

maybe a game aggregate building up from many commands

Team

Add a recap of all the Commands and Events
Then show how each one builds up an aggregate

Dispatch(RegTeam{id: 123, name: "A"})
* The router will look for the pid of a GenServer {:aggregate, Team, 123}* in the Registry (Not there)
* Spawn a new GenServer named {:aggregate, Team, 123}*
* Send it the command as a message
* Allow & Emit TeamReg Event
* It will be stored in the EventStore for us in the stream "team_123"*

* Only true in concept

Add Player 1
* Locate GenServer {:aggregate, Team, 123} in the Registry
* It's there, send it the command as a message
* Allow & Emity PlayerJoinedTeamEvent
* Appended to the Event store as the 2nd event in the stream "team_123"

*Kick over server*
Add player 1
* Lookup {:aggregate, Team, 123}* in the Registry
* It's not there, spawn a new GenServer named {:aggregate, Team, 123}*
* load all the events in EventStore for stream "team_123"
* apply them 1 by 1, changing the GenServer's state
* Reject the command as Player 1 is already on the team.


~3/4~ flat view of stack of commands filling up a number of game and team aggregates

register team a
add player 1 to team a
register team b
add player 2 to team b
add player 3 to team b
schedule game a for team a
schedule game b for team b
cancel game a
start game a => fail
add player 1 to teama (again) => fail


TeamRegistered{name: "A")
PlayerJoinedTeam{
TeamRegistered
register team b
add player 2 to team b
add player 3 to team b
schedule game a for team a
schedule game b for team b
cancel game a



NOW you are ready to show them the code version.

We've already seen the commands and events
But they look like this:
Show 1 command and 1 Event on the same slide
try to pick one where the event contains some extra info from the state

---

Now show the aggregate in skeletal form with the struct, two execs and two apply funcs

--

now show the game aggregate with the validation

--

From here, they have a conceptual grounding in the aggregate, command, events and their roles.
Recap, talking about accepting all the writes, but the system can't show anybody anything, time for read side

Show Event Handler for projections
Show Event Handler for side effects (email)
Show That Event Handlers are a single named genserver

Recap the pattern so far, we can repeat this all day but something is missing

Show Process manager
layout the problem
graphic
implemtnation
tak about the lifecycle and how each PM instance is a GenServer

Recap?
Commands are sent to Aggregates, which validate them and emit events
Events are stored in the EventStore in streams which correspond to aggregates
EventHandlers perform sideeffect and create projections
ProcessManagers coordinate between aggregates behind the scenes



---

class: center, middle

# Ben Moss
### Bitfield Consulting

[ben@bitfield.co](mailto:ben@bitfield.co)

[github.com/drteeth](https://github.com/drteeth)

[@benjamintmoss](https://twitter.com/benjamintmoss)

???

Hi I’m Ben Moss,
* I consultant doing Ruby, Elixir and Android work through my shop Bitfield

---
class: center, middle

# Event sourcing/CQRS in Elixir with Commanded:
## Where the rubber hits the road.

???

This is my 2nd talk on ES/CQRS, the last one was theoretical and this will be about how to apply the patterns in Elixir.

---
class: center, middle, inverse

# Recap
Last talk:
https://drteeth.github.io/elixir-es-cqrs

???

There are a number of talks out there describing both ES and CQRS, including one of my own so I will try to recap the patterns very quick and move on to the implementation details.

---
class: center, middle

# Event Sourcing
Building up state from a series of events that describe facts and mutations.

???

This is the idea that instead of recording the current state of a model, we instead record the history of mutations against it and build up the state of the model from there. We'll go into more detail soon.

---
class: center, middle

# CQRS
### Command Query Responsibility Segregation
Separate the read and write models.

???

The main idea here is that there is code that models writing new data and then there is code that handles reads and that they can evolve separately.

We'll see more on that shortly.

---
class:

# The pieces:
![Event Sourcing Diagram](images/event_sourcing_overview_0.svg)

???

These are the main components in our system. We'll cover each in detail as we go through the code.

---
class:

# The Project: Attend
* Attendance tracker for my pickup soccer league.
* Schedules games for teams
* Allow teams to ask their members for attendance to a game.
* Members can reply by clicking a link in their email.

???

To go through the example, I'll go over an attendance tracker that I wrote for a soccer leage I play with. The idea is that for each game, the app will send out an email to each player asking if they will attend, and they can respond yes/no/maybe with a click.

It's quite simple, but it will take us through some interesting aspects of the patterns.

---
class: middle, center

Demo time

???

Register a team
Add a player
Schedule a game
Start a Check
Answer a Check

---
class: middle, center, inverse

# The Write Side

???

First I'll cover the write side.

---
class:

# An acceptance test to work from

```elixir
# Register a new team
team_id = Attend.register_team("The Alchemists")

# Add a player
Attend.add_player_to_team(team_id, "Mattia", "mattia@example.com")

# Schedule a game for that team
game_id = Attend.schedule_game(team_id, "Blah Park", @game_time)

# Check the attendance for the game and that team
Attend.check_attendance(game_id, team_id)

# Simulate clicking "Yes" in the email sent to Mattia
{player_check_id, yes_token} = parse_token(last_email(), "Yes")
Attend.confirm_attendance(player_check_id, yes_token)
```

???

You can imagine working from an acceptance test something like this.
* Register a team
* Add a player
* Schedule a game
* Check attendance
* Confirm attendance

---
class:

# An acceptance test to work from

```elixir
# Schedule a game for that team
game_id = Attend.schedule_game(team_id, "Blah Park", @game_time)
```


---
class:
# The API, Command and Router

* Commands are structs
* The router dispatches commands for us.
* Nothing is returned. We don't read here, this is the write side.

```elixir
defmodule Attend
  def schedule_game(team_id, location, start_time) do
  game_id = UUID.generate()

  command = %ScheduleGame{
    game_id: game_id,
    team_id: team_id,
    location: location,
    start_time: start_time
  }

  case Router.dispatch(command) do
    :ok ->
      {:ok, game_id}

    error ->
      error
  end
end
```

???

We'll pickup in the middle where we're scheduling a game for a team.
We create a new instance of a command, which are structs, and dispatch it with the Router which we'll see next.

We generate and return the ID here but often people will generate the ID on the client side and just return "OK".

---
class:

# The Command

![Event Sourcing Diagram](images/event_sourcing_overview_1.svg)

---
class:

# The Command

* It’s a serializable struct.
* Named in the imperative

```elixir
defmodule Attend.Commands.ScheduleGame do
  @derive Jason.Encoder
  defstruct [:game_id, :team_id, :location, :start_time]
end
```

???

It's a struct that represents the command. Here we're want to schedule a game.

---
class:
# The Router

* Maps commands to aggregates
* Specifies an indentity
* Will find or create a GenServer by aggregate type & identity

```elixir
defmodule Attend.CommandRouter do
  dispatch(
    [
      ScheduleGame,
      CheckAttendance,
      StartGame,
      CancelGame,
      EndGame
    ],
    to: Game,
    identity: :game_id
  )
end
```

???

* Here we map the ScheduleGame command to the Game.
* Commanded will spawn a GenServer based on the identity and module type given.
* That means we're going to create a new aggregate and spawn a GenServer to listen for commands on, and then we're going to send it the command to handle.

---
class:

# The Aggregate

* Is a module with a struct
* Is backed by a GenServer and spawned lazily
* Allows serial access to an instance of an aggregate

```elixir
defmodule Attend.Aggregates.Game do
  defstruct [
    :game_id,
    :location,
    :team_id,
    :start_time,
    :status
  ]
```
???

The aggregate is a struct representing the current state for the purposes of validating commands against. This is the model of our game on the write side.

---
class:

# The Aggregate
### (as Command Handler)

![Event Sourcing Diagram](images/event_sourcing_overview_4.svg)

---
class:

# The Aggregate
### (as Command Handler)

* Returns an error tuple or 0 or more events
* Written to the event store for us

```elixir
  def execute(%Game{} = _game, %ScheduleGame{} = command) do
    %GameScheduled{
      game_id: command.game_id,
      team_id: command.team_id,
      location: command.location,
      start_time: command.start_time
    } # Will be written to the event store
  end
```

???

This is the spot where we decide if we're going to accept or reject a command. If we accept it, we'll emit 1 or more events which describe the change in the system.

If we reject it, we'll return an error we can show to the user.

This is the first Command in the Game's lifecycle and we're returning a GameScheduled event.

---
class:

# The Event

![Event Sourcing Diagram](images/event_sourcing_overview_2.svg)

---
class:
# Event

* It’s a serializable struct
* Named in past tense
* Represents a fact about our system
* Often the paired with a Command (ScheduleGame => GameScheduled)

```elixir
defmodule Attend.Events.GameScheduled do
  @derive Jason.Encoder
  defstruct [:game_id, :team_id, :location, :start_time]
end
```

???

Not much to say here yet other than these are the gold in your system. They enable everything else.

---
class:

# The Aggregate
### (applying events for new state)

* Apply clause for each event on an aggregate
* Takes the previous state and an event as input
* Returns the new state
* Applied after the events have been persisted
* Also applied when re-hydrating the aggregate


```elixir
  def apply(%Game{} = game, %GameScheduled{} = event) do
    %{ game |
        game_id: event.game_id,
        team_id: event.team_id,
        location: event.location,
        start_time: event.start_time,
        status: :scheduled
    }
  end
```

???

Here we're setting up the initial state of our Game so we can validate further commands against it later.

---
# The Aggregate
### (a 2nd command)

* Validate input based on state
* Protect invariants
* Our oportunity to give feedback to the user

```elixir
  def execute(%Game{} = game, %CheckAttendance{} = command) do
    cond do
      game.status == :cancelled ->
        {:error, :game_cancelled}

      game.status == :ended ->
        {:error, :game_already_ended}

      game.team_id != command.team_id ->
        {:error, :team_not_scheduled_for_game}

      true ->
        %AttendanceCheckStarted{
          check_id: command.check_id,
          game_id: command.game_id,
          team_id: command.team_id
        }
    end
  end
```

???

Here we're handling another command against our aggregate. We only want to allow an attendance check if the game scheduled. We also want to provide meaningful feedback to the user if we reject the command.

---
# The Aggregate

```elixir
defmodule Attend.Aggregates.Game do
  defstruct [
    :game_id,
    :location,
    :team_id,
    :start_time,
    :status
  ]

  def execute(%Game{} = game, %ScheduleGame{} = command) do
    %GameScheduled{...}
  end

  def execute(%Game{} = game, %CheckAttendance{} = command) do
    %AttendanceCheckStarted{...}
  end

  def apply(%Game{} = game, %GameScheduled{} = event) do
   %{game | status: :scheduled}
  end

  def apply(%Game{} = game, %CheckAttendance{} = event) do
    ...
  end
```

???

Here is what our Game aggregate looks like

---
class:

# The story so far:

* Maintain invariants with aggregate
* There's 1 per ID/Type: Game(123), Game(234), Team(345)
* Aggregates are running concurently with each other
* Aggregates run serially internally
* Express and capture a series of intents
* Transform those into immutable facts

???

---
class: middle, center, inverse

# The Read Side

???

We can leave the write side for now and see how we get data out of the system. Queries are handled by the read side.

---
class:

# Projections / Event Handlers

![Event Sourcing Diagram](images/event_sourcing_overview_5.svg)

???

The read side is a series of what is called projections of the events into an cohesive whole that we can query on.

---
class:

# Projections / Event Handlers

* Populate caches/tables to handle queries from the UI
* Built on the events from the Write side
* 1 GenServer per Projection
* Listens for Events and performs effects
* Writing to a cache, sending an email, etc

???

---

# TeamDetail projection

```elixir
defmodule Attend.EventHandlers.TeamProjector do
  use Commanded.Event.Handler, name: __MODULE__

  def handle(%Events.TeamRegistered{} = event, _metadata) do
    TeamDetail.insert(%Team{
      id: event.team_id,
      name: event.name,
      players: []
    })
    :ok
  end
```

???

Here we have the Team detail projection. It's job is to listen for all of the events that it need to produce a blob of detail that the UI can use to fill out the detail view of the team. That means: Upcoming games, team members, name etc.

We start off by listening for the Team Register even and persist that to disk. Storage is not prescribed, I'm using redis here but I've hidden the details away behind a module.

---


# TeamDetail projection

```elixir
defmodule Attend.EventHandlers.TeamProjector do
  use Commanded.Event.Handler, name: __MODULE__

  def handle(%Events.TeamRegistered{} = event, _metadata) do
    TeamDetail.insert(%Team{
      id: event.team_id,
      name: event.name,
      players: []
    })
    :ok
  end

  def handle(%Events.JoinedTeam{} = event, _metadata) do
    TeamDetail.get(event.team_id)
    |> TeamDetail.add_player(event.player)
    :ok
  end

  def handle(%Events.LeftTeam{} = event, _metadata) do
    TeamDetail.get(event.team_id)
    |> TeamDetail.remove_player(event.player)
    :ok
  end
```

???

For subsequent team events, we can update the state

---

# TeamDetail projection

```elixir
  def handle(%Events.GameScheduled{} = event, _metadata) do
    Team.add_game(%Game{
      game_id: event.game_id,
      location: event.location,
      start_time: event.start_time,
      team_id: event.team_id
    })
    :ok
  end

  def handle(%Events.GameCancelled{} = _event, _metadata) do
     # Mark the game cancelled
    :ok
  end

  def handle(%Events.GameEnded{} = _event, _metadata) do
     # Mark the game ended
    :ok
  end
```

???

Event handlers can use events from across aggregates to build up their state

---
class: inverse, center, middle

# Sending the attendance email

---

# Sending the attendance email

```elixir
Attend.check_attendance(game_id, team_id)
```

---

# Sending the attendance email

```elixir
Attend.check_attendance(game_id, team_id)
```

* A game must have been scheduled
* An attedance check must have been requested
* Use an Event handler to listen for games to be scheduled
* And send an email when attendance is asked for a player on a team in a game.

---

# Sending the attendance email

```elixir
# Check the attendance for the game and that team
Attend.check_attendance(game_id, team_id)
```

* A game must have been scheduled
* An attedance check must have been requested
---
![Event Sourcing Diagram](images/attend_emailer_01.svg)

???

---
![Event Sourcing Diagram](images/attend_emailer_02.svg)

???

---
![Event Sourcing Diagram](images/attend_emailer_03.svg)

???

---
![Event Sourcing Diagram](images/attend_emailer_04.svg)

???

---
![Event Sourcing Diagram](images/attend_emailer_05.svg)

???

---
![Event Sourcing Diagram](images/attend_emailer_06.svg)

???

---
![Event Sourcing Diagram](images/attend_emailer_07.svg)

???

---
![Event Sourcing Diagram](images/attend_emailer_08.svg)

???

---
![Event Sourcing Diagram](images/attend_emailer_09.svg)

???

---

# A pattern emerges:
* Register team
* Add player
* Remove player
* Cancel game
* Start game
* End game

???

We've seen enough of the basic now that we can continue this way for a long while.
All of these features follow this basic pattern more or less.
We can continue to add projections and handlers as we've seen.
There's a problem though.

We can see it when we go to request attendance for a team's players to a game.

---

# Attendance Check

### Requirements
* Don't allow a *Check* to be requested for a *Game* that has ended or been cancelled.
* Ask each player from a given *Team* if they will be attending a given Game.
* Wait to hear from each player

???

We've got all the data we need, but we can't cross aggregate boundaries to get at it.

---

# Attendance Check

### Requirements
* Don't allow a *Check* to be requested for a *Game* that has ended or been cancelled.
* Ask each player from a given *Team* if they will be attending a given Game.
* Wait to hear from each player

### Which aggregate will handle the command?

???

---

# Attendance Check

### Requirements
* Don't allow a *Check* to be requested for a *Game* that has ended or been cancelled.
* Ask each player from a given *Team* if they will be attending a given Game.
* Wait to hear from each player

### Which aggregate will handle the command?
1. We need to validate against the status of Game, so Game?

???

---

# Attendance Check

### Requirements
* Don't allow a *Check* to be requested for a *Game* that has ended or been cancelled.
* Ask each player from a given *Team* if they will be attending a given Game.
* Wait to hear from each player

### Which aggregate will handle the command?
1. We need to validate against the status of Game, so Game?
2. We need the list of players that Team has, so Team?

???

---

# Attendance Check

### Requirements
* Don't allow a *Check* to be requested for a *Game* that has ended or been cancelled.
* Ask each player from a given *Team* if they will be attending a given Game.
* Wait to hear from each player

### Which aggregate will handle the command?
1. We need to validate against the status of Game, so Game?
2. We need the list of players that Team has, so Team?
3. It feels like we'd like to model the Check as it's own aggregate, so Check?

???

---
# Process Manager

![Event Sourcing Diagram](images/event_sourcing_overview_6.svg)

???

The Process Manager is our missing piece. They live downstream of the persisted events, but can feed new commands back into the system.

---

# Process Manager

* Listens for events
* Can issue new commands
* Can aggregate state
* Is a GenServer spawned dynamically on an initial event
* *Must* be keyed on a common key (every event must contain that id)
* Has a lifecycle based on events

???

---
![Event Sourcing Diagram](images/attend_process_manager_01.svg)

???

---
![Event Sourcing Diagram](images/attend_process_manager_02.svg)

???

---
![Event Sourcing Diagram](images/attend_process_manager_03.svg)

???

---
![Event Sourcing Diagram](images/attend_process_manager_04.svg)

???

---
![Event Sourcing Diagram](images/attend_process_manager_05a.svg)

???

---
![Event Sourcing Diagram](images/attend_process_manager_05b.svg)

???

---
![Event Sourcing Diagram](images/attend_process_manager_06.svg)

???

---
![Event Sourcing Diagram](images/attend_process_manager_07.svg)

???

---
![Event Sourcing Diagram](images/attend_process_manager_08.svg)

???

---
![Event Sourcing Diagram](images/attend_process_manager_09.svg)

???

---
![Event Sourcing Diagram](images/attend_process_manager_10.svg)

???

---
![Event Sourcing Diagram](images/attend_process_manager_11.svg)

???

---

![Event Sourcing Diagram](images/attend_process_manager_12.svg)

???

---


https://github.com/commanded/commanded
https://github.com/slashdotdash


Benefits:
The core of your app really does work with out the web. Very little to do.
