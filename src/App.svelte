<script>
  import { gql } from "apollo-boost";
  import ApolloClient from "apollo-client";
  import { WebSocketLink } from "apollo-link-ws";
  import { HttpLink } from "apollo-link-http";
  import { getMainDefinition } from "apollo-utilities";
  import { split } from "apollo-link";
  import { InMemoryCache } from "apollo-cache-inmemory";
  import { query, mutate, subscribe } from "svelte-apollo";

  const server = "k7o9c.sse.codesandbox.io";

  const wsLink = new WebSocketLink({
    uri: `wss://${server}/graphql`,
    options: {
      reconnect: true
    }
  });

  const httpLink = new HttpLink({
    uri: `https://${server}/graphql`
  });

  const link = split(
    ({ query }) => {
      const { kind, operation } = getMainDefinition(query);
      return kind === "OperationDefinition" && operation === "subscription";
    },
    wsLink,
    httpLink
  );

  const noteQuery = gql`
    query {
      note {
        from
        message
      }
    }
  `;

  const noteMutation = gql`
    mutation($from: String, $message: String) {
      updateNote(from: $from, message: $message) {
        from
        message
      }
    }
  `;

  const noteSubscription = gql`
    subscription {
      noteUpdated {
        from
        message
      }
    }
  `;

  const client = new ApolloClient({
    link,
    cache: new InMemoryCache()
  });

  const note = query(client, { query: noteQuery });
  const noteUpdates = subscribe(client, { query: noteSubscription });

  noteUpdates.subscribe(console.log);
</script>

<style>
  .container {
    display: flex;
    flex-wrap: wrap;
  }
</style>

<button
  on:click={() => mutate(client, {
      mutation: noteMutation,
      variables: { from: 'Mindy' }
    })} />

{#await $note}
  Loading...
{:then result}
   {result.data.note.from} {result.data.note.message}
{:catch error}
  Error: {error}
{/await}

{#await $noteUpdates}
  Waiting for updates...
{:then result}
   {JSON.stringify(result)}
{:catch error}
   {JSON.stringify(error)}
{/await}
