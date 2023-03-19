---
title: "Foundry 101"
date: 2023-03-12T11:15:29+01:00
---

Until some time the only way to poke around with EVM smart contracts development was to use tools written in JavaScript (truffle, hardhat) or Python (brownie) - unfortunately for me, not a fan of those languages, it was a bit harsh. :)

Luckily, over the last year, a new kid in the block has grown, it's called Foundry and I really liked it.

## Building blocks

Foundry is not one tool, it's a set of tools, a toolkit that nicely follows [Unix philosophy](https://en.wikipedia.org/wiki/Unix_philosophy) - programs doing one thing, doing it well and playing sane with each other.

The tools are:
- forge - helping us with build -> test -> deploy cycle,
- anvil - serving as a local testnet node,
- cast - utility for "talking" to Ethereum node,
- chisel - [REPL](https://en.wikipedia.org/wiki/Read–eval–print_loop) environment for solidity.

Let's briefly take a look at their purposes and usage.

## Forge

Forge is the foundation of Foundry, it lets you manage, build and test your code.  

From now on, we're going to leverage the source code that is generated by default with a new project and see what happens there.  

### Project layout

To create a new project it's either:

`$ mkdir counter; cd counter; forge init`

or if you're in the right directory already just: 

`$ forge init counter` 

Here's what we got:

```
$ tree -L 2

├── foundry.toml
├── lib
│ └── forge-std
├── script
│ └── Counter.s.sol
├── src
│ └── Counter.sol
└── test
  └── Counter.t.sol
```

### Config

`foundry.toml` is a configuration file, for our use case it's good to just know it's there - we won't use anything other than defaults. 

For more details on various toggles, check [this doc](https://github.com/foundry-rs/foundry/tree/master/config).

### Dependencies

`lib/` is where dependencies live. By default, forge creates a new git repository and dependencies are handled as git submodules.  

To add a new dependency from GitHub:  
`forge install transmissions11/solmate`.  

If you need more, `forge install -h` is your friend.

To avoid long import paths in the code, forge offers remappings feature.  
Let me just serve you an example:

```
$ forge remappings

ds-test/=lib/forge-std/lib/ds-test/src/
forge-std/=lib/forge-std/src/
solmate/=lib/solmate/src/
```

Here are the remappings that forge inferred automatically.  
If you want to add more, then all remappings need to be stored in `remappings.txt` (there is a way to store them in foundry.toml, alternatively)


From now on, you can use statements like  
 `import "solmate/src/tokens/ERC20.sol";`  
 and forge will automatically resolve `solmate` into `lib/solmate/src/` when needed.

### Building the project

Forge generated some code for us in `src/Counter.sol`, let's take a look.

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.17;

contract Counter {
    uint256 public number;

    function setNumber(uint256 newNumber) public {
        number = newNumber;
    }

    function increment() public {
        number++;
    }
}
```

To build the code, run `forge build`. 

Btw. there is also `forge fmt` subcommand that comes in handy when you add more code and want it nicely formatted.

As you can see, the code is simple, but it serves well in our baby steps. It creates a contract with one public variable and gives us two functions to modify it - let's play with them by running tests.


### Testing

The great thing about Foundry is that its tools feel very "native", for example, a test in foundry is just another smart contract that interacts with our code.  

Here's a part of generated `test/Counter.t.sol`:

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.17;

import "forge-std/Test.sol";
import "../src/Counter.sol";

contract CounterTest is Test {
    Counter public counter;

    function setUp() public {
        counter = new Counter();
        counter.setNumber(0);
    }

    function testIncrement() public {
        counter.increment();
        assertEq(counter.number(), 1);
    }

    // [...]
}
```

Let's dig more into some specific lines:

- `contract CounterTest is Test` - we need to inherit Test contract.  
Thanks to that we can make assertions, alter the state of EVM during a test and so on.  
Speaking about altering the state of EVM - [cheatcodes](https://book.getfoundry.sh/forge/cheatcodes) are used for that, I won't cover them in this post, though.
- `Counter public counter;` creates an instance of our unit being tested.
- `setUp()` is called before each test case.  
It's supposed to set the desired state of things before the test case runs.
- `testIncrement()` is a test case.  
Every function prefixed with `test` is treated by Forge as a test case. 
- `assertEq(counter.number(), 1);` checks if the output matches the expected value.

To run a specific test by name run:

```
$ forge test -vvvv --match-test testIncrement

Running 1 test for test/Counter.t.sol:CounterTest
[PASS] testIncrement() (gas: 28334)
Traces:
 [28334] CounterTest::testIncrement() 
 ├─ [22340] Counter::increment() 
 │ └─ ← ()
 ├─ [283] Counter::number() [staticcall]
 │ └─ ← 1
 └─ ← ()

Test result: ok. 1 passed; 0 failed; finished in 258.04µs
```

The result gives hints about the gas units usage (in square brackets) and the call stack.

Foundry generated one more test case for us - it presents [fuzz testing](https://en.wikipedia.org/wiki/Fuzzing). 

```
    function testSetNumber(uint256 x) public {
        counter.setNumber(x);
        assertEq(counter.number(), x);
    }
```

Notice the value passed as a parameter - it will be mutated.  
In our case, the only parameter is `uint256 x` but nothing stops you from defining more.

Let's run it:

```
forge test -vvvv --match-test testSetNumber

Running 1 test for test/Counter.t.sol:CounterTest
[PASS] testSetNumber(uint256) (runs: 256, μ: 27476, ~: 28409)
Traces:
 [28409] CounterTest::testSetNumber(3) 
 ├─ [22290] Counter::setNumber(3) 
 │ └─ ← ()
 ├─ [283] Counter::number() [staticcall]
 │ └─ ← 3
 └─ ← ()

Test result: ok. 1 passed; 0 failed; finished in 9.63ms
```

Notice the `runs: 256` - it means our test case was run with 256 mutations of type uint256.  
Tidle ("~") on the other hand informs us on the median gas used across all runs.

### Deployment

OK, we have the code, it's tested, and now we'd like to deploy it to the network. How do we do it?

There are two ways. 

For simple contracts, we can just use `create` subcommand.

```
$ forge create \
    --rpc-url http://127.0.0.1:8545 \
    --private-key 0xac097... \
    src/Counter.sol:Counter
```

The output is pretty compact: 

```
Deployer: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
Deployed to: 0x5FbDB2315678afecb367f032d93F642f64180aa3
Transaction hash: 0x8e8c5e09385bf91b3
```

If there are multiple contracts to be deployed at once or lots of arguments to be passed in constructors - it's better to use Foundry's scripts.

Forge already created an example script for Counter contract, but it's incomplete. 
We need to mark when the broadcasting starts and stops and initialize our contract in beetwen.

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Script.sol";
import "../src/Counter.sol";

contract CounterScript is Script {
    function setUp() public {}

    function run() public {
        vm.startBroadcast();
        Counter counter = new Counter();
        vm.stopBroadcast();
    }
}

```

The command to run is:

```
forge script \
    --broadcast \
    --verify \
    --rpc-url http://127.0.0.1:8545 \
    --private-key 0xac0974... \
    script/Counter.s.sol:CounterScript
```

The output now is more verbose, it even contains gas cost calcations in ETH:

```
Estimated gas price: 3.689158116 gwei

Estimated total gas used for script: 138734

Estimated amount required: 0.000511811662065144 ETH
```

Also, detailed transaction dumps are stored in `./broadcase/<script name>/<chain id>/`, in our case: `./broadcast/Counter.s.sol/31337/run-latest.json`

However, to deploy our code one way or another, we need to learn a bit about Anvil first. 

## Anvil

Anvil is super simple to use but so much helpful. It runs a local Ethereum node and generates a number of test accounts with just one command. 

Another great feature is the ability to fork an existing network. That means you can simulate the same state, for example, as the latest block on Ethereum mainnet and interact with contracts that live on the chain - an efficent way to check how your code interacts with other protocols.  

The only thing needed is the URL of the archive node (also that could be Infura or Alchemy API) passed as `--fork-url`.
It's called fork because every change made to the state afterwards stays in our local environment.  

Here's an example of an instance that forks Ethereum mainnet at the latest block and creates two test accounts:

```
anvil -a 2 --fork-url https://eth-mainnet.g.alchemy.com/v2/<key>

                             _   _
                            (_) | |
      __ _   _ __   __   __  _  | |
     / _` | | '_ \  \ \ / / | | | |
    | (_| | | | | |  \ V /  | | | |
     \__,_| |_| |_|   \_/   |_| |_|

    0.1.0 (5c2db0b 2023-01-31T00:14:55.309941Z)
    https://github.com/foundry-rs/foundry

Available Accounts
==================

(0) "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266" (10000 ETH)
(1) "0x70997970C51812dc3A010C7d01b50e0d17dc79C8" (10000 ETH)

Private Keys
==================

(0) 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
(1) 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d

[...]

Fork
==================
Endpoint: https://eth-mainnet.g.alchemy.com/v2/<key>
Block number: 16792699
Block hash: 0xa8312cdd77b576095d70d6cb6f5b3cfc80848fc83a256001800f9dd5afcfd060
Chain ID: 1


Listening on 127.0.0.1:8545
```

Having the instance running, let's use Cast to interact with it.

## Cast

Cast is a swiss army knife of Foundry. 
It lets you query the state of the blockchain, send transactions, perform different kinds of conversions and even do [ENS](https://ens.domains) lookups - but again, for more details `cast -h` is your friend. ;) 

Great thing is that all foundry tools support passing `--rpc-url` / `--fork-url` parameter (naming is a bit inconsistent though) so something like below is possible:
```
$ cast resolve-name --rpc-url https://eth-mainnet.g.alchemy.com/v2/<key> seblw.eth

0xDa8d58b89A0c00877e0912b893a47E947A02Aa32
```

Cool, right? And there is so much more.  
I'll present more cast use cases in the last part.

## Chisel

Chisel is pretty new in Foundry family and honestly, I haven't played with it much.

You can think of it the same way as typing `python` in your commandline - write code and receive instant feedback.

Here's something just to give you an idea of how it works:

```
~ ❯ chisel
Welcome to Chisel! Type `!help` to show available commands.
➜ uint256 public number;
➜ number = 1337;
➜ number
Type: uint
├ Hex: 0x539
└ Decimal: 1337
➜ number = number + 1;
➜ number
Type: uint
├ Hex: 0x53a
└ Decimal: 1338
```

## Piecing things together

Let's see how Foundry tools play together and simulate a simple development workflow.

First, let's run our anvil instance.

```
$ anvil
```

Next, we'll deploy Counter contract via `forge create` as follows:

```
$ forge create \
    --rpc-url http://127.0.0.1:8545 \
    --private-key 0xac0974... \
    src/Counter.sol:Counter
```

And finally, we can interact with it on chain using cast.

Let's use `send` subcommand first to invoke `increment()` function (thus send a transaction).

```
$ cast send <addr> "increment()" --private-key 0xac097...

blockHash               0xdcfbdfc7055ac86c5c89ccbe3bf1ce69f0bae8abc68fa2168e8167a94bccba6b
blockNumber             2
contractAddress
cumulativeGasUsed       43404
effectiveGasPrice       3875889325
gasUsed                 43404
logs                    []
logsBloom               0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
000000000000000000
root
status                  1
transactionHash         0x69e75fda4b45e0a5deaa7e3f00df415857a6d58c677266db416b1454697c68d9
transactionIndex        0
type
```

Then use `call` subcommand to query `number()` getter (no transaction).

```
$ cast call <addr> "number()"

0x0000000000000000000000000000000000000000000000000000000000000001
```

Once again `send`, this time with `setNumber(uint256)` signature and an argument.

```
$ cast send --private-key 0xac097... <addr> "setNumber(uint256)" 0x1337 
$ cast call <addr> "number()"

0x0000000000000000000000000000000000000000000000000000000000001337
```

See the 1337 there? It means our job for today is done.