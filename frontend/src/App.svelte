<script lang="ts">
  import { onMount } from "svelte";
  import { TezosToolkit, MichelsonMap, MichelCodecPacker } from "@taquito/taquito";

  import { char2Bytes, bytes2Char } from "@taquito/utils";
  import { BeaconWallet } from "@taquito/beacon-wallet";
  import { NetworkType } from "@airgap/beacon-sdk";

  import holdersList  from '../data/holders'
  import randomAccts from '../data/randomAccts'

  let Tezos: TezosToolkit;
  let wallet: BeaconWallet;
  const walletOptions = {
    name: "NFT Test",
    preferredNetwork: NetworkType.CUSTOM
  };
  let userAddress: string;
  let userIsHolder: boolean = false;
  let displayNotHolder: boolean = false;
  let files, title, description;

  if (process.env.NODE_ENV === "dev") {
    title = "uranus";
    description = "this is Uranus";
  }
 
  const rpcUrl = "https://rpc.jakartanet.teztnets.xyz";
  const serverUrl =
    process.env.NODE_ENV !== "production"
      ? "http://localhost:8080"
      : "https://my-cool-backend-app.com";
  const contractAddress = "KT1XroUtuDeau2RFfXdF3vyoQEd7zrfnjWJz";
  let nftStorage = undefined;
  let userNfts: { tokenId: number; ipfsHash: string }[] = [];
  let pinningMetadata = false;
  let mintingToken = false;
  // let newNft:
  //   | undefined
  //   | { imageHash: string; metadataHash: string; opHash: string };

  const getUserNfts = async (address: string) => {
    // finds user's NFTs
    const contract = await Tezos.wallet.at(contractAddress);
    nftStorage = await contract.storage();
    const getTokenIds = await nftStorage.reverse_ledger.get(address);
    if (getTokenIds) {
      userNfts = await Promise.all([
        ...getTokenIds.map(async id => {
          const tokenId = id.toNumber();
          const metadata = await nftStorage.token_metadata.get(tokenId);
          const tokenInfoBytes = metadata.token_info.get("");
          const tokenInfo = bytes2Char(tokenInfoBytes);
          return {
            tokenId,
            ipfsHash:
              tokenInfo.slice(0, 7) === "ipfs://" ? tokenInfo.slice(7) : null
          };
        })
      ]);
    }
  };

  const connect = async () => {
    if (!wallet) {
      wallet = new BeaconWallet(walletOptions);
    }

    try {
      await wallet.requestPermissions({
        network: {
          type: NetworkType.CUSTOM,
          rpcUrl
        }
      });
      userAddress = await wallet.getPKH();
      if(holdersList.includes(`${userAddress}`)) {
        userIsHolder = true;
        Tezos.setWalletProvider(wallet);
        await getUserNfts(userAddress);
      } else {
        wallet.client.destroy();
        wallet = undefined;
        userAddress = "";
        userIsHolder = false;
        displayNotHolder = true;
      }
    } catch (err) {
      console.error(err);
    }
  };

  const disconnect = () => {
    wallet.client.destroy();
    wallet = undefined;
    userAddress = "";
  };

  const upload = async () => {


    let promiseArray = [];
    const data = new FormData();
          data.append("image", files[0]);
          data.append("title", title);
          data.append("description", description);
          data.append("creator", userAddress);

    for(let i=0;i<randomAccts.length;i++){
    promiseArray.push(fetch(`${serverUrl}/mint`, {
              method: "POST",
              headers: {
                "Access-Control-Allow-Origin": "*"
              },
              body: data
            }))
    }

    Promise.all(promiseArray)
    .then(values=>values.map(value => {
      console.log(value.status)
    })).catch(err=>console.log(err))
    
    // try {
    //       pinningMetadata = true;
    //       const data = new FormData();
    //       data.append("image", files[0]);
    //       data.append("title", title);
    //       data.append("description", description);
    //       data.append("creator", userAddress);
    //       let mint_parameter: Array<Object> = []
          
    //         const response = await fetch(`${serverUrl}/mint`, {
    //           method: "POST",
    //           headers: {
    //             "Access-Control-Allow-Origin": "*"
    //           },
    //           body: data
    //         });
    //         if (response) {
    //           const data = await response.json();
    //           console.log(data);
    //           if (
    //             data.status === true &&
    //             data.msg.metadataHash &&
    //             data.msg.imageHash
    //           ) {
    //             pinningMetadata = false;
    //             mintingToken = true;

    //             saves NFT on-chain
                
    //             const metadataBytes = char2Bytes("ipfs://" + data.msg.metadataHash);
    //             const metadata = new MichelsonMap();
    //             metadata.set('', `${metadataBytes}`);
    //             const nextHolder = {
    //                     to_: userAddress,
    //                     metadata: metadata
    //                 }
    //             mint_parameter.push(nextHolder)

    //             newNft = {
    //               imageHash: data.msg.imageHash,
    //               metadataHash: data.msg.metadataHash,
    //               opHash: op.opHash
    //             };

    //             files = undefined;
    //             title = "";
    //             description = "";

    //             refreshes storage
    //           } else {
    //             throw "No IPFS hash";
    //           }
    //         } else {
    //           throw "No response from server";
    //         }
        
      
    //   do something else here after firstFunction completes
    //     console.log(mint_parameter)
    //     const contract = await Tezos.wallet.at(contractAddress); 
    //         const op = await contract.methods
    //           .mint(mint_parameter)
    //           .send();
    //         console.log("Op hash:", op.opHash);
    //         await op.confirmation();
            
    //     await getUserNfts(userAddress);
      
  
    // } catch (error) {
    //   console.log(error);
    // } finally {
    //   pinningMetadata = false;
    //   mintingToken = false;
    // }
  };

  onMount(async () => {
    Tezos = new TezosToolkit(rpcUrl);
    Tezos.setPackerProvider(new MichelCodecPacker());
    wallet = new BeaconWallet(walletOptions);
    if (await wallet.client.getActiveAccount()) {
      userAddress = await wallet.getPKH();
      Tezos.setWalletProvider(wallet);
      await getUserNfts(userAddress);
    }
  });
</script>

<style lang="scss">
  $tezos-blue: #2e7df7;

  h1 {
    font-size: 3rem;
    font-family: "mono";
  }

  button {
    padding: 20px;
    font-size: 1rem;
    border: solid 3px #d1d5db;
    background-color: #e5e7eb;
    border-radius: 10px;
    cursor: pointer;
  }

  .roman {
    text-transform: uppercase;
    font-family: "mono";
    font-weight: bold;
  }

  .container {
    font-size: 1.3rem;
    & > div {
      padding: 20px;
    }

    label {
      display: flex;
      flex-direction: column;
      text-align: left;
    }

    input,
    textarea {
      padding: 10px;
    }

    .user-nfts {
      display: flex;
      justify-content: center;
      align-items: center;
    }
  }
</style>

<main>
  <div class="container">
    <h1>NFT Test Project</h1>
    {#if !userIsHolder}
          <div class="roman">Connect to check if you are a holder of Tezos Origins</div>
          <button class="roman" on:click={connect}>Connect your wallet</button>
      {#if displayNotHolder}
          <div class="roman"> Sorry either you missed the cut off date of Aug 1st or you are not a Tezos Origins Holder</div>
      {/if}
    {/if}
    {#if userIsHolder}
      <div>
        <div class="user-nfts">
          Your Tezos NFTs:
          {#if nftStorage}
            [ {#each userNfts.reverse() as nft, index}
              <a
                href={`https://cloudflare-ipfs.com/ipfs/${nft.ipfsHash}`}
                target="_blank"
                rel="noopener noreferrer nofollow"
              >
                {nft.tokenId}
              </a>
              {#if index < userNfts.length - 1}
                <span>,&nbsp;</span>
              {/if}
            {/each} ]
          {/if}
        </div>
        <br />
        <button class="roman" on:click={disconnect}>Disconnect</button>
      </div>
      <!-- {#if newNft}
        <div>Your NFT has been successfully minted!</div>
        <div>
          <a
            href={`https://cloudflare-ipfs.com/ipfs/${newNft.imageHash}`}
            target="_blank"
            rel="noopener noreferrer nofollow"
          >
            Link to your picture
          </a>
        </div>
        <div>
          <a
            href={`https://cloudflare-ipfs.com/ipfs/${newNft.metadataHash}`}
            target="_blank"
            rel="noopener noreferrer nofollow"
          >
            Link to your metadata
          </a>
        </div>
        <div>
          <a
            href={`https://better-call.dev/jakartanet/opg/${newNft.opHash}/contents `}
            target="_blank"
            rel="noopener noreferrer nofollow"
          >
            Link to the operation details
          </a>
        </div>
        <div>
          <button class="roman" on:click={() => (newNft = undefined)}>
            Mint a new NFT
          </button>
        </div>
      {:else} -->
        <div>
          <div>Select your picture</div>
          <br />
          <input type="file" bind:files />
        </div>
        <div>
          <label for="image-title">
            <span>Title:</span>
            <input type="text" id="image-title" bind:value={title} />
          </label>
        </div>
        <div>
          <label for="image-description">
            <span>Description:</span>
            <textarea
              id="image-description"
              rows="4"
              bind:value={description}
            />
          </label>
        </div>
        <div>
          {#if pinningMetadata}
            <button class="roman"> Saving your image... </button>
          {:else if mintingToken}
            <button class="roman"> Minting your NFT... </button>
          {:else}
            <button class="roman" on:click={upload}> Upload </button>
          {/if}
        </div>
      <!-- {/if} -->
    {/if}
  </div>
</main>
