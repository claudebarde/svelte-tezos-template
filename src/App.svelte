<script lang="ts">
  import { onMount, onDestroy } from "svelte";
  import { TezosToolkit } from "@taquito/taquito";
  import { BeaconWallet } from "@taquito/beacon-wallet";
  import { NetworkType } from "@airgap/beacon-sdk";
  import { HubConnectionBuilder, HubConnection } from "@microsoft/signalr";

  let Tezos: TezosToolkit;
  let wallet: BeaconWallet;
  let subscription: HubConnection;
  let blockHead: { protocol: string; level: number; lastUpdate: string };

  const rpcUrl = "https://api.tez.ie/rpc/mainnet";
  const packages: { name: string; display: string; version: number }[] = [
    { name: "svelte", display: "Svelte", version: 3 },
    { name: "webpack", display: "Webpack", version: 5 },
    { name: "typescript", display: "TypeScript", version: 3 },
    { name: "sass", display: "Sass", version: 5 },
    { name: "taquito", display: "Taquito", version: 8 },
    { name: "beacon", display: "Beacon SDK", version: 2 }
  ];

  const connect = async () => {
    try {
      wallet = new BeaconWallet({
        name: "Svelte Tezos Template",
        preferredNetwork: NetworkType.MAINNET
      });
      await wallet.requestPermissions({
        network: {
          type: NetworkType.MAINNET,
          rpcUrl
        }
      });
      Tezos.setWalletProvider(wallet);
    } catch (err) {
      console.error(err);
    }
  };

  const disconnect = () => {
    wallet.client.destroy();
    wallet = undefined;
  };

  const subscribeToEvents = async () => {
    const connection = new HubConnectionBuilder()
      .withUrl("https://api.tzkt.io/v1/events")
      .build();

    // auto-reconnect
    connection.onclose(subscribeToEvents);

    connection.on("head", msg => {
      blockHead = {
        protocol: msg.data.protocol,
        level: msg.data.level,
        lastUpdate: msg.data.timestamp
      };
    });

    // open connection
    await connection.start();
    // subscribe to head
    await connection.invoke("SubscribeToHead");
    // return connection instance
    return connection;
  };

  onMount(async () => {
    Tezos = new TezosToolkit(rpcUrl);
    const headerInfo = await Tezos.rpc.getBlockHeader();
    blockHead = {
      protocol: headerInfo.protocol,
      level: headerInfo.level,
      lastUpdate: headerInfo.timestamp
    };
    subscription = await subscribeToEvents();
  });

  onDestroy(async () => {
    // closes subscription when component unmounts
    await subscription.stop();
  });
</script>

<style lang="scss">
  $tezos-blue: #2e7df7;

  .container {
    font-size: 20px;
    max-width: 50%;

    .title {
      color: $tezos-blue;
      font-size: 100px;
      margin: 20px;
    }

    .subtitle {
      font-size: 30px;
      color: #333;
      margin: 10px;
    }

    ul {
      width: 30%;
      margin: 0 auto;
      text-align: left;
      list-style: none;

      li {
        display: flex;
        justify-content: left;
        align-items: center;
        padding: 3px;

        img {
          width: 30px;
          height: 30px;
          margin-right: 20px;
        }
      }
    }

    .chain-info {
      font-size: 15px;

      p {
        margin: 5px;
        font-style: italic;
      }
    }

    button {
      appearance: none;
      border: solid 2px $tezos-blue;
      border-radius: 5px;
      background-color: white;
      padding: 20px;
      font-size: 20px;
      color: $tezos-blue;
      transition: 0.3s;
      cursor: pointer;
      outline: none;

      &:hover {
        color: white;
        background-color: $tezos-blue;
      }
    }
  }
</style>

<main>
  <div class="container">
    <div class="title">Hello Tezos!</div>
    <div class="subtitle">
      A Svelte template to build awesome dapps on Tezos!
    </div>
    <br />
    {#if blockHead}
      <div class="chain-info">
        <p>Protocol: {blockHead.protocol}</p>
        <p>Level: {blockHead.level}</p>
        <p>Block timestamp: {blockHead.lastUpdate}</p>
      </div>
    {/if}
    <br />
    <div>
      This template comes with all the good stuff you need to start building a
      dapp on Tezos:
    </div>
    <br /><br />
    <ul>
      {#each packages as pkg}
        <li>
          <img src={`images/${pkg.name}.png`} alt={pkg.name} />
          {pkg.display} v.{pkg.version}
        </li>
      {/each}
    </ul>
    <br /><br />
    <div>
      {#if wallet}
        <button on:click={disconnect}>Disconnect</button>
      {:else}
        <button on:click={connect}>Connect now!</button>
      {/if}
    </div>
  </div>
</main>
