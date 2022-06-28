# Siblings    [![Kantox ❤ OSS](https://img.shields.io/badge/❤-kantox_oss-informational.svg)](https://kantox.com/)  [![Test](https://github.com/am-kantox/siblings/workflows/Test/badge.svg)](https://github.com/am-kantox/siblings/actions?query=workflow%3ATest)  [![Dialyzer](https://github.com/am-kantox/siblings/workflows/Dialyzer/badge.svg)](https://github.com/am-kantox/siblings/actions?query=workflow%3ADialyzer)

**The partitioned dynamic supervision of FSM-backed workers.**

## Usage

`Siblings` is a library to painlessly manage many uniform processes,
all having the lifecycle _and_ the _FSM_ behind.

Consider the srtvice, that polls the market rates from several
diffferent sources, allowing semi-automated trading based
on predefined conditions. For each bid, the process is to be spawn,
polling the external resources. Once the bid condition is met,
the bid gets traded.

With `Siblings`, one should implement `c:Siblings.Worker.perform/3`
callback, doind actual work and returning either `:ok` if no action
should be taken, or `{:transition, event, payload}` to initiate the
_FSM_ transition. When the _FSM_ get exhausted (reaches its end state,)
both the performing process _and_ the _FSM_ itself do shut down.

_FSM_ instances leverage [`Finitomata`](https://hexdocs.pm/finitomata)
library, which should be used alone if no recurrent `perform` should be
accomplished _or_ if the instances are not uniform.

Typical code for the `Siblings.Worker` implementation would be as follows

```elixir
defmodule MyApp.Worker do
  @fsm """
  born --> |reject| rejected
  born --> |bid| traded
  """

  use Finitomata, @fsm

  def on_transition(:born, :reject, _nil, payload) do
    perform_rejection(payload)
    {:ok, :rejected, payload}
  end

  def on_transition(:born, :bid, _nil, payload) do
    perform_bidding(payload)
    {:ok, :traded, payload}
  end

  @behaviour Siblings.Worker

  @impl Siblings.Worker
  def perform(state, id, payload)

  def perform(:born, id, payload) do
    cond do
      time_to_bid?() -> {:transition, :bid, nil}
      stale?() -> {:transition, :reject, nil}
      true -> :ok
    end
  end

  def perform(:rejected, id, _payload) do
    Logger.info("The bid #{id} was rejected")
    {:transition, :__end__, nil}
  end

  def perform(:traded, id, _payload) do
    Logger.info("The bid #{id} was traded")
    {:transition, :__end__, nil}
  end
end
```

Now it can be used as shown below

```elixir
{:ok, pid} = Siblings.start_link()
Siblings.start_child(MyApp.Worker, "Bid1", %{}, interval: 1_000)
Siblings.start_child(MyApp.Worker, "Bid2", %{}, interval: 1_000)
...
```

The above would spawn two processes, checking the conditions once
per a second (`interval`,) and manipulating the underlying _FSM_ to
walk through the bids’ lifecycles.

_Sidenote:_ Normally, `Siblings` supervisor would be put into 
the supervision tree of the target application.

## Installation

```elixir
def deps do
  [
    {:siblings, "~> 0.1"}
  ]
end
```

## Changelog

* `0.2.0` — Fast `Worker` lookup
* `0.1.0` — Initial MVP

## [Documentation](https://hexdocs.pm/siblings)
