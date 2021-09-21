# Injecting and discovering dependencies in Supervision trees

## Our task

We need siblings of supervisors to find out about each other. Preferably we also want our supervisor tree to be hermetic: the possibility to start multiple instances of them with different options.

Example: a pipeline for processing network data which consists of several intercommunicating processes that start and consume data. We would like to start different pipelines in an ad-hoc fashion for different data sources so they should not interfere and be hermetic.

There are several ways to do it.

## Our options

### Hardcoded process name registration

The easiest and most common way is to register your processes using global names

```elixir
defmodule PipelineSupervisor do
  use Supervisor
  
  @impl Supervisor
  def init([]) do
    children = [
      {SomeConsumer, name: ConsumerGlobalName}
      {SomeWorker, %{consumer: ConsumerGlobalName}},
    ]

    Supervisor.init(children, strategy: :rest_for_one)
  end
end

defmodule SomeWorker do
  use GenServer

  @impl GenServer
  def init(%{consumer: consumer} = options) do
    {:ok, options}
  end

  @impl GenServer
  def handle_call({:process_data_chunk, data_chunk}, _from, state) do
    {:reply, :ok, do_process_data_chunk(data_chunk, state)}
  end

  defp do_process_data_chunk(data_chunk, state) do
    case parse_data_somehow(data_chunk, state) do
      {:ok, parsed_message, new_state} ->
        Process.send(state.consumer, {:message_received, parsed_message})
        new_state

      {:more_data_necessary, new_state} ->
        new_state
    end
  end
end

defmodule SomeConsumer do
  use GenServer

  def start_link(%{name: name}) do
    GenServer.start_link(__MODULE__, [], name: name)
  end

  @impl GenServer
  def init([]) do
    {:ok, :undefined_state}
  end

  @impl GenServer
  def handle_info({:message_received, parsed_message}, state) do
    # ...
    {:noreply, state}
  end
end
```

Instead of implicit name registration provided by erlang, you can use some process registry that will serve basically as DI container: a way to register and lookup dependencies (process pid) by some arbitrary id.

These include: 
* Using a well-known and reliable registry, for instance, Elixir.Registry
* Using some self-written registry that stores its state (process pid mappings) as GenServer State or in ETS

When the process starts, it registers itself. When its pid is required by other processes they ask this DI registry for necessary dependency.

But the problem is still here. It is a chicken and egg dilemma: To call registry, you need to know its pid or name.

This solution is easy and common, but it prevents us from running the supervision tree multiple times because of the hardcoded process name of Consumer.

### Prefixed process name registration

To make our supervision tree hermetic, we can prefix all processes with some common key and make it an init parameter of our core supervisor.

```elixir
defmodule PipelineSupervisor do
  use Supervisor
  
  @impl Supervisor
  def init(%{name_prefix: prefix}) do
    consumer_name = Module.concat([__MODULE__, prefix, Consumer])
    
    children = [
      {SomeConsumer, name: consumer_name}
      {SomeWorker, %{consumer: consumer_name}},
    ]

    Supervisor.init(children, strategy: :rest_for_one)
  end
end
```

Although it gets the job done, it clutters the process namespace and creates API that might seem awkward ("why do you need me to provide some prefix?").

### Implicit registry using Supervisor.which_children function

If we pay closer attention to Supervisor we can notice that any supervisor is a registry itself. It naturally knows pids of all its processes and all processes have unique ids local to the supervisor.

Unfortunately (or luckily) Supervisors are single-purpose building blocks and therefore lack convenient apis to lookup their children. However, we still have the way to use supervisors as registers using  [`Supervisor.which_children`](https://hexdocs.pm/elixir/1.12/Supervisor.html#which_children/1) function.


```elixir
defmodule PipelineSupervisor do
  use Supervisor
  
  @impl Supervisor
  def init([]) do
    supervisor_pid = self()

    children = [
      SomeConsumer,
      {SomeWorker, %{parent_supervisor: supervisor_pid}},
    ]

    Supervisor.init(children, strategy: :rest_for_one)
  end

  def child_pid!(supervisor_pid, child_id) do
    spec = Supervisor.which_children(supervisor_pid)

    case for {^id, child, _type, _modules} <- spec, do: child do
      [child_pid] when is_pid(child_pid) ->
        child_pid

      _else ->
        raise "no started child with id:#{id} in supervisor spec:#{inspect(spec)}"
    end
  end
end

defmodule SomeWorker do
  use GenServer

  @impl GenServer
  def init(%{supervisor_pid: supervisor_pid} = options) do
    {:ok, options}
  end

  defp do_process_data_chunk(data_chunk, state) do
    # ...
    consumer_pid = PipelineSupervisor.child_pid!(state.supervisor_pid, SomeConsumer)
    Process.send(consumer_pid, {:message_received, parsed_message})
    # ...
    end
  end
end

defmodule SomeConsumer do
  use GenServer

  def start_link([]) do
    GenServer.start_link(__MODULE__, [])
  end

end
```

In some cases calling the supervisor on each message might create a bottleneck. You can fix this by calling `child_pid!` once in `Worker.init` function and saving `Consumer` pid in its state.
In this case, you should be careful and think about what would happen if `Consumer` crashes. In our case, everything will be fine because of the [`rest_for_one`](https://hexdocs.pm/elixir/1.12/Supervisor.html#module-strategies) supervisor strategy.

### Starting dependencies ad-hoc

Also, we can exploit the fact that Supervisor returns pid of started process in `Supervisor.start_child` function.

```elixir
defmodule PipelineSupervisor do
  use Supervisor
  
  @impl Supervisor
  def init([]) do
    supervisor_pid = self()

    children = [
      {SomeWorker, %{parent_supervisor: supervisor_pid}},
    ]

    Supervisor.init(children, strategy: :rest_for_one)
  end

  def start_consumer do
    Supervisor.start_child(SomeConsumer)
  end
end

defmodule SomeWorker do
  use GenServer

  @impl GenServer
  def init(%{supervisor_pid: supervisor_pid} = options) do
    {:ok, options, {:continue, :start_worker}}
  end

  def handle_continue(:start_worker, state) do
    {:ok, consumer_pid} = PipelineSupervisor.start_consumer()
    {:noreply, Map.put(state, :consumer_pid, consumer_pid)}
  end

  defp do_process_data_chunk(data_chunk, state) do
    # ...
    Process.send(state.consumer_pid, {:message_received, parsed_message})
    # ...
    end
  end
end

defmodule SomeConsumer do
  use GenServer

  def start_link([]) do
    GenServer.start_link(__MODULE__, [])
  end

end
```

Note that we have to use `:handle_continue` callback to avoid deadlocking the supervisor. Otherwise, it would not be able to reply to our `start_child` call because it would be waiting `init` function to return.

This method of DI is convoluted and probably not worth it just for the sake of itself. But it can make more sense in more complicated scenarios where `SomeWorker` instead of just sending messages to `Consumer` also needs to manage its lifetime (kill or restart). 

## Conslusion

Some of the options that I showed may seem complex and unnecessary. And in many cases that is entirely true. But my point is that using OTP is much more than just writing GenServers and occasionally some custom Supervisor or two. OTP gives us a pretty basic but very smart combination of base elements that we can use to form some higher-level usage patterns. Our task as a community is to discover those patterns and make them simple to use.
