<script lang="ts">
  import { Aptos, AptosConfig, Network } from "@aptos-labs/ts-sdk";
  import { CheckCircle, Circle, Plus, Wallet } from "lucide-svelte";

  type Todo = {
    id: number;
    title: string;
    description: string;
    completed: boolean;
    editing: boolean;
  };

  const config = new AptosConfig({ network: Network.TESTNET });
  const aptos = new Aptos(config);
  const CONTRACT_ADDRESS =
    "0x3ae7aa633fe2bcbcf9a03791b701f5b73170e55cd2ac60ebcfcb290aa59c1c4d";
  const MODULE = "tasks";

  let tasks: Todo[] = $state([]);
  let userAddress = $state("");
  let isConnected = $state(false);
  let isInitialized = $state(false);

  let todoBody = $state({
    title: "",
    description: "",
    completed: false,
  });

  const connectWallet = async () => {
    try {
      // @ts-ignore
      if (window.aptos) {
        // @ts-ignore
        const response = await window.aptos.connect();
        if (response) {
          isConnected = true;
          userAddress = response.address;

          await getAllTasks();
        }
      }
    } catch (error) {
      isConnected = false;
      console.log("error", error);
    }
  };

  const disconnectWallet = async () => {
    try {
      // @ts-ignore
      if (window.aptos) {
        // @ts-ignore
        await window.aptos.disconnect();
        isConnected = false;
        isInitialized = false;

        tasks = [];
      }
    } catch (error) {
      console.log("error", error);
    }
  };

  const checkExistingConnection = async () => {
    try {
      //@ts-ignore
      if (window.aptos && window.aptos.isConnected()) {
        //@ts-ignore
        const account = await window.aptos.account();
        isConnected = true;
        userAddress = account.address;
      }
    } catch (error) {
      console.log("error", error);
    }
  };

  const checkInitialization = async () => {
    try {
      //@ts-ignore
      const result = await window.aptos.signAndSubmitTransaction({
        payload: {
          type: "entry_function_payload",
          function: `${CONTRACT_ADDRESS}::${MODULE}::initialize_task_list`,
          arguments: [],
          type_arguments: [],
        },
      });

      await aptos.waitForTransaction({ transactionHash: result.hash });

      if (result.length > 0) {
        isInitialized = true;
        // @ts-ignore
        tasks = result[0];
      }
    } catch (error) {
      // @ts-ignore
      if (error.message && error.message.includes("MISSING_DATA")) {
        isInitialized = false;
        tasks = [];
      }
    }
  };

  const getAllTasks = async () => {
    try {
      // @ts-ignore
      const account = await window.aptos.account();
      const payload = {
        function: `${CONTRACT_ADDRESS}::${MODULE}::get_all_tasks`,
        functionArguments: [account.address],
      };
      // @ts-ignore
      const result = await aptos.view({ payload });

      if (result.length > 0) {
        // @ts-ignore
        tasks = result[0];
        isInitialized = true;
      }

      console.log("result", result);
    } catch (error) {
      tasks = [];
      isInitialized = false;
      console.log("error", error);
    }
  };

  const get_task = async (id: number) => {
    try {
      // @ts-ignore
      const account = await window.aptos.account();
      // Asegurarse de que TaskList esté inicializado
      // await this.initializeIfNeeded();
      const payload = {
        function: `${CONTRACT_ADDRESS}::${MODULE}::get_task`,
        functionArguments: [account.address, id],
      };
      // @ts-ignore
      const result = await aptos.view({ payload });
      console.log("result", result);
    } catch (error) {
      console.log("error", error);
    }
  };

  const create_task = async (title: string, description: string) => {
    try {
      if (!isInitialized) {
        await checkInitialization();
      }

      // @ts-ignore
      const response = await window.aptos.signAndSubmitTransaction({
        payload: {
          type: "entry_function_payload",
          function: `${CONTRACT_ADDRESS}::${MODULE}::create_task`,
          type_arguments: [],
          arguments: [title, description],
        },
      });

      await aptos.waitForTransaction({ transactionHash: response.hash });

      if (response) {
        getAllTasks();

        todoBody = {
          title: "",
          description: "",
          completed: false,
        };
      }

      console.log("Tarea creada con éxito");
    } catch (error) {
      console.log("error", error);
    }
  };

  const editTask = async (
    id: number,
    title: string,
    description: string,
    completed: boolean
  ) => {
    try {
      //@ts-ignore
      const response = await window.aptos.signAndSubmitTransaction({
        payload: {
          type: "entry_function_payload",
          function: `${CONTRACT_ADDRESS}::${MODULE}::edit_task`,
          type_arguments: [],
          arguments: [id, title, description, completed],
        },
      });

      await aptos.waitForTransaction({ transactionHash: response.hash });

      tasks = tasks.map((task) => {
        if (task.id === id) {
          task.title = title;
          task.description = description;
          task.completed = completed;
          task.editing = false;
        }
        return task;
      });

      console.log("response", response);
    } catch (error) {
      console.log("error", error);
    }
  };

  const toggleTodo = async (id: number) => {
    try {
      const task = tasks.find((t) => t.id === id);
      // if (task) task.completed = !task.completed;

      if (task) {
        task.completed = !task.completed;
        const response = editTask(
          id,
          task.title,
          task.description,
          task.completed
        );

        tasks = tasks.map((task) => {
          if (task.id === id) {
            task.completed = !task.completed;
          }
          return task;
        });
        console.log("response", response);
      }
    } catch (error) {
      console.log("error", error);
    }
  };

  const startEdit = (id: number) => {
    const task = tasks.find((t) => t.id === id);
    if (task) task.editing = true;
  };

  const saveEdit = (id: number, title: string, description: string) => {
    const task = tasks.find((t) => t.id === id);
    if (task && title.trim() && description.trim()) {
      task.title = title;
      task.description = description;
      task.editing = false;
    }
  };

  const cancelEdit = (id: number) => {
    const task = tasks.find((t) => t.id === id);
    if (task) task.editing = false;
  };

  const deleteTodo = async (id: number) => {
    // tasks = tasks.filter((t) => t.id !== id);
    try {
      //  @ts-ignore
      const response = await window.aptos.signAndSubmitTransaction({
        payload: {
          type: "entry_function_payload",
          function: `${CONTRACT_ADDRESS}::${MODULE}::delete_task`,
          type_arguments: [],
          arguments: [id],
        },
      });

      aptos.waitForTransaction({ transactionHash: response.hash });

      if (response) {
        tasks = tasks.filter((t) => t.id !== id);
        console.log("response", response);
      }
    } catch (error) {
      console.log("error", error);
    }
  };

  (() => {
    checkExistingConnection();
    // checkInitialization();
    getAllTasks();
  })();
</script>

<div class="min-h-screen bg-gradient-to-br from-slate-100 to-slate-200 p-4">
  <div class="mx-auto max-w-4xl">
    <!-- Header -->

    <div class="mb-8 text-center">
      <h1 class="mb-4 text-4xl font-bold text-slate-800">
        Blockchain Aptos Todo App
      </h1>
      <p class="text-slate-600">Manage your tasks on the Aptos blockchain</p>
    </div>

    <!-- Wallet Connection -->

    <div class="mb-8 rounded-xl bg-white p-6 shadow-lg">
      <div class="flex items-center justify-between">
        <div class="flex items-center gap-3">
          <Wallet class="h-6 w-6 text-slate-600" />
          <span class="text-lg font-medium text-slate-800">
            {isConnected
              ? `Wallet Connected to ${userAddress.includes("...") ? userAddress : userAddress.slice(0, 6) + "..." + userAddress.slice(-6)}`
              : "Wallet Status"}
          </span>
        </div>

        <div class="flex gap-3s">
          {#if !isConnected}
            <button
              onclick={connectWallet}
              class="flex items-center gap-2 rounded-lg bg-blue-600 px-4 py-2 text-white transition-colors hover:bg-blue-700 cursor-pointer"
            >
              <Wallet class="h-4 w-4" />
              Connect Wallet</button
            >
          {:else}
            <div class="flex items-center gap-3">
              <span
                class="rounded-full bg-green-100 px-3 py-1 text-sm font-medium text-green-800"
              >
                Connected
              </span>
              <button
                onclick={disconnectWallet}
                class="rounded-lg bg-slate-600 px-4 py-2 text-white transition-colors hover:bg-slate-700 cursor-pointer"
              >
                Disconnect
              </button>
            </div>
          {/if}
        </div>
      </div>
    </div>

    <!-- Add New Todo -->
    <div class="mb-8 rounded-xl bg-white p-6 shadow-lg">
      <h2 class="mb-4 text-xl font-semibold text-slate-800">Add New Todo</h2>
      <form
        onsubmit={(e) => {
          e.preventDefault();
          create_task(todoBody.title, todoBody.description);
          // checkInitialization();
        }}
        class="flex gap-3"
      >
        <input
          bind:value={todoBody.title}
          type="text"
          placeholder="Title..."
          class="flex-1 rounded-lg border border-slate-300 px-4 py-2 focus:border-blue-500 focus:outline-none focus:ring-2 focus:ring-blue-200"
          disabled={!isConnected}
        />

        <input
          bind:value={todoBody.description}
          type="text"
          placeholder="Description..."
          class="flex-1 rounded-lg border border-slate-300 px-4 py-2 focus:border-blue-500 focus:outline-none focus:ring-2 focus:ring-blue-200"
          disabled={!isConnected}
        />

        <button
          type="submit"
          disabled={!isConnected ||
            !todoBody.title.trim() ||
            !todoBody.description.trim()}
          class="flex items-center gap-2 rounded-lg bg-green-600 px-6 py-2 text-white transition-colors hover:bg-green-700 disabled:bg-gray-400 disabled:cursor-not-allowed"
        >
          <Plus class="h-4 w-4" />
          Add Todo
        </button>
      </form>
      {#if !isConnected}
        <p class="mt-2 text-sm text-amber-600">
          Connect your wallet to add todos
        </p>
      {/if}
    </div>

    <!-- Todos List -->

    <div class="rounded-xl bg-white shadow-lg">
      <div class="border-b border-slate-200 p-6">
        <h2 class="text-xl font-semibold text-slate-800">Your Todos</h2>
        <p class="text-slate-600">
          {tasks.filter((t) => !t.completed).length} pending, {tasks.filter(
            (t) => t.completed
          ).length} completed
        </p>
      </div>

      {#if tasks.length === 0}
        <div class="p-12 text-center">
          <Circle class="mx-auto mb-4 h-12 w-12 text-slate-400" />
          <p class="text-slate-500">No tasks yet. Add your first todo above!</p>
        </div>
      {:else}
        <div class="divide-y divide-slate-200">
          {#each tasks as todo (todo.id)}
            <div class="todo-item p-4 transition-colors hover:bg-slate-50">
              <div class="flex items-center justify-between">
                <div class="flex items-center gap-3">
                  <!-- Status Icon -->
                  <button
                    onclick={() => toggleTodo(todo.id)}
                    class="flex-shrink-0 transition-colors"
                    disabled={!isConnected}
                  >
                    {#if todo.completed}
                      <CheckCircle
                        class="h-6 w-6 text-green-600 cursor-pointer"
                      />
                    {:else}
                      <Circle
                        class="h-6 w-6 text-slate-400 hover:text-slate-600 cursor-pointer"
                      />
                    {/if}
                  </button>

                  <!-- Todo Content -->
                  <div class="flex-1">
                    {#if todo.editing}
                      <form
                        onsubmit={(e) => {
                          e.preventDefault();
                        }}
                        class="flex gap-2"
                      >
                        <input
                          name="title"
                          value={todo.title}
                          class="flex-1 rounded border border-slate-300 px-2 py-1 focus:border-blue-500 focus:outline-none"
                        />

                        <input
                          name="description"
                          value={todo.description}
                          class="flex-1 rounded border border-slate-300 px-2 py-1 focus:border-blue-500 focus:outline-none"
                        />
                      </form>
                    {:else}
                      <div>
                        <span class="text-sm text-slate-500">ID: {todo.id}</span
                        >
                        <h3
                          class="font-medium text-slate-800 {todo.completed
                            ? 'line-through opacity-60'
                            : ''}"
                        >
                          {todo.title}
                        </h3>

                        <p
                          class="text-sm text-slate-600 {todo.completed
                            ? 'line-through opacity-60'
                            : ''}"
                        >
                          {todo.description}
                        </p>
                      </div>
                    {/if}
                  </div>
                </div>

                <!-- Actions moved here for better visibility -->
                {#if isConnected}
                  <div class="flex gap-2">
                    {#if !todo.editing}
                      <button
                        class="rounded-lg px-3 py-1 text-sm font-medium transition-colors {todo.completed
                          ? 'bg-yellow-100 text-yellow-800 hover:bg-yellow-200'
                          : 'bg-green-100 text-green-800 hover:bg-green-200'}"
                      >
                        {todo.completed ? "Undo" : "Complete"}
                      </button>
                      <button
                        onclick={() => startEdit(todo.id)}
                        class="rounded-lg bg-blue-100 px-3 py-1 text-sm font-medium text-blue-800 transition-colors hover:bg-blue-200 cursor-pointer"
                        disabled={todo.completed}
                      >
                        Edit
                      </button>
                      <button
                        onclick={() => deleteTodo(todo.id)}
                        class="rounded-lg bg-red-100 px-3 py-1 text-sm font-medium text-red-800 transition-colors hover:bg-red-200 cursor-pointer"
                      >
                        Delete
                      </button>
                    {:else}
                      <button
                        onclick={(e) => {
                          e.preventDefault();
                          const todoItem = (e.target as Element).closest(
                            ".todo-item"
                          );
                          if (todoItem) {
                            const form = todoItem.querySelector("form");
                            if (form) {
                              const formData = new FormData(form);
                              const title = formData.get("title") as string;
                              const description = formData.get(
                                "description"
                              ) as string;

                              editTask(
                                todo.id,
                                title,
                                description,
                                todo.completed
                              );
                            }
                          }
                        }}
                        type="submit"
                        class="rounded-lg bg-green-600 px-3 py-1 text-sm font-medium text-white hover:bg-green-700 cursor-pointer"
                      >
                        Save
                      </button>
                      <button
                        type="button"
                        onclick={() => cancelEdit(todo.id)}
                        class="rounded-lg bg-slate-600 px-3 py-1 text-sm font-medium text-white hover:bg-slate-700 cursor-pointer"
                      >
                        Cancel
                      </button>
                    {/if}
                  </div>
                {/if}
              </div>
            </div>
          {/each}
        </div>
      {/if}
    </div>
  </div>
</div>
