# zkSNARKs: Driver's Ed.
## A practical beginner's guide to creating, proving, and verifying zkSNARKs in your contracts
Joseph Stockermans 
Jan. 6th 2018

![enter image description here](https://images.unsplash.com/photo-1499102443154-09b49b31a97b?auto=format&fit=crop&w=1500&q=80)

## Motivation
Like many people, when I first heard about the Byzantium update a few months ago, I was super excited for all the new privacy features that zkSNARKS would offer us - without really understanding anything about zkSNARKs at all. Nevertheless I was eager to write some contracts with secret values and stuff like that as soon as I got a chance. 

I recently took some time to explore the subject and my journey started with [this article from Consensys](https://media.consensys.net/introduction-to-zksnarks-with-examples-3283b554fc3b) written by Christian Lundkvist. It gives an excellent layman's explanation of zkSNARKs and drafts a token contract with hidden balances (if you have not read this article I highly recommend you do as I will refer to it a lot going forward). I thought this was very cool and decided to try to recreate this contract on my own. The contract code contains a mysterious function called `zksnarkverify`. I wasn't exactly sure what this was or what was included in the Byzantium update, so I initially assumed that this was some kind of new built-in solidity function to use zkSNARKs. Unfortunately, as it turns out, this function was just a vision of a function that *could* exist, and the actual implementation was left to the reader to create for themselves. 

I started Googling to find someone's implemetation of this function and I was quickly overwhelmed by the difficulty of understanding most of the popular articles out there on the subject: Christian Reitwiessner's [zkSNARKs in a Nutshell](https://blog.ethereum.org/2016/12/05/zksnarks-in-a-nutshell/), Vitalik Buterin's 3-part series [Zk-SNARKs: Under the Hood](https://medium.com/@VitalikButerin/zk-snarks-under-the-hood-b33151a013f6), and this seemingly impenetrable [Solidity contract](https://gist.github.com/chriseth/f9be9d9391efc5beb9704255a8e2989d) that stackoverflow users were claiming to be ["the zksnarks solidity code"](https://ethereum.stackexchange.com/questions/16473/is-there-any-working-zk-snark-implementation-even-if-experimental-among-the-ex/21229). These all seemed very complicated to me and none of them really discussed getting up and running with zkSNARKs.  In fact many commenters on the "zksnarks contract" gist seemed to have no idea where to get started.

What I was looking for was a copy+paste from stackoverflow type answer. When I finally figured something out, I decided to write this tutorial for developers who just want to know how to drive the zkSNARKs car, and not how the car works - so I hope you find this tutorial helpful!



## Starting the Engine
### Introduction to ZoKrates

I remembered a project I had heard about somewhere called [ZoKrates](https://github.com/JacobEberhardt/ZoKrates) that had something to do with Ethereum and zkSNARKs. It turns out that ZoKrates is awesome, and is so far the only way I could find that allows you to create and verify a zero-knowledge proof in a Solidity contract, without needing to know much about what is actually going on "under-the-hood". ZoKrates claims to be a "tool box for zkSNARKs on Ethereum". What I found, is that ZoKrates is a way to create and run all those `C`, `G`, `P`, and `V` functions found in the Consensys article that make up a zkSNARK. On top of this, as it turns out, you can't just write a zero-knowledge proof program as a Javascript function like the article illustrates. Programs written for verification with a zero-knowledge proof need to be written as something called an Arithmetic Circuit. I still don't really know exactly what arithmetic circuits are, but ZoKrates has a built-in higher level language that compiles simple python-like scripts into arithmetic circuits for you! So much thanks to Jacob Eberhardt and all the ZoKrates contributors for building this and helping me in my zkSNARKs journey!

### Getting Started
The easiest way to get started with zkSNARKs and ZoKrates is to work with Docker. 

Make sure you have docker installed and running.

    # Clone the repo
    $ git clone https://github.com/JacobEberhardt/ZoKrates.git
    $ cd ZoKrates
    
    # Build the Docker image
    $ docker build -t zokrates .
    
    # Run a zokrates container and jump inside
    $ docker run -ti zokrates /bin/bash
    $ cd ZoKrates

Great! Now we're inside a Docker container that has all the ZoKrates commands available to us. Before we run anything, I'm going to provide a little description for each of the ZoKrates commands found on their github README, since there is currently no explanation for them.

 - `compile`
	 - Takes a file written in the ZoKrates higher level language and compiles it into an arithmetic circuit (creates the arithmetic circuit `C` from the Consensys article).
	 - Mandatory arguments
		 - `-i path/to/file.code`
 - `setup`
	 - Generates the proving key and verification key from the arithemetic circuit and "toxic-waste" parameter `lambda` (runs the generator function from the Consensys article `(pk, vk) = G(λ, C)`). 
 - `compute-witness `
	 - Creates a "witness" for use in generating a proof. A proof is dependent on specific values of public and private arguments. These values are called the witness.
	 - Mandatory arguments
		 - `-a <arg1> <arg2> ... <argn>`
 - `generate-proof`
	 - Creates a proof that is dependent on a witness and a proving-key (runs the proving function `π = P(pk, x, w)` from the Consensys article).
 - `export-verifier`
		 - Creates a Solidity contract file `verifier.sol` that contains a hardcoded verifier-key and a public function `verifyTx`. This public function accepts a proof and a public input. When `verifyTx` is called it runs a version of the verifier function `V(vk, x, π)` from the Consensys article, AKA the mysterious `zksnarkverify` function!

Now that we have a reference for what the ZoKrates command-line can do, let's build, prove, and verifying a zkSNARK on the Ethereum blockchain!

### Who runs which functions and when?

To explain how all these ZoKrates functions fit together, lets assume three entities: (1) Charlie, a trusted 3rd party who generates and then destroys the "toxic waste" parameter `lambda`, (2) Alice, a user interested in proving something to the Ethereum community without revealing all her secrets, and (3) Bob, an observer watching transactions on the Ethereum blockchain.

#### Charlie (a trusted third party)
First, Charlie comes up with a "program" (arithmetic circuit) that he thinks will be useful to people. He creates a useful program that takes public and private input. He runs the generator function `G`, destroys `lambda`  and shares the proving-key `pk` and verification-key `vk` with the Ethereum community. Finally, he creates a Solidity contract with the verification-key `vk` hardcoded inside of it. This contract also has a public function that users can submit proofs and inputs to, called `verifyTx`.

#### Alice (the contract user)
Next up is Alice. She thinks Charlie's program is very useful and wants to use it to prove something to the Ethereum community without revealing all the values contributing to her proof. She computes a witness that fulfills Charlie's program. She then runs the proving function `P`, inputting her witness and the publicly available proving-key `pk`. She sends the resulting proof along with her public input to Charlie's function `verifyTx`. Since Alice did everything correct, the proof is verified and the `verifyTx` outputs `true`.

#### Bob (an observer)
Meanwhile, Bob has been watching Charlie's contract. He see's that Alice has sent a transaction to the `verifyTx` function, and that this function returned `true`. He can be certain that this function will only output `true` if Alice provided a proof generated from Charlie's proving-key `pk` and if Alice's public and private input fulfill the necessary constraints of Charlie's program. In other words, he can be sure Alice's proof is valid while still knowing nothing about her private inputs into the proof.

## Taking zkSNARKs for a Test Drive
### A concrete example
#### Charlie (a trusted third party)
First let's watch which ZoKrates commands Charlie, the trusted party, runs. Charlie thinks that it would be beneficial to the Etheruem community to have a contract in which users can prove that they know a set of three numbers that sum to `15`. He believes that this type of proof would benefit from some privacy so he designs a program that accepts one public and two private inputs and asserts that the sum of the three is equal to `15`.

Charlie's program looks something like this, and assuming we are in the ZoKrates root directory, he saves it to a file called 
`sumsToFifteen.code`.

    def main(x):
	    s1 + s2 + x == 15
	    return 1

In Charlie's program, `x` is the public input, and `s1` and `s2` are secret values *(ZoKrates is currently specified so that secret values are not not explicitly passed to the function. Bear in mind that ZoKrates is a new project and a WIP so many details may be subject to change in the future)*. The second line of the program is an assertion statement. Users will not be able to generate proofs for this program if values passed to it cause the the assertion to fail. On the third line, we see that if a proof constructed from this program is verified, it will output `1`.

Charlie compiles this program written in the ZoKrates higher level language into an arithmetic circuit with the `compile` command.

    $ ./target/release/zokrates compile -i sumsToFifteen.code

This outputs a machine-readable circuit file called `out` and a human readable circuit file called `out.code`. Charlie then runs the generator function `G`, giving us a prover key `pk` and verification key `vk`.

    $ ./target/release/zokrates setup

This command creates a `verifiation.key` file and a `proving.key` file. Charlie makes these files publicly available to the Ethereum community. He then creates a Solidity contract and hardcodes the verification key inside.

    $ ./target/release/zokrates export-verifier

This command creates a `verifier.sol` Solidity contract file that contains our verification key and a handy function called `verifyTx`. Charlie deploys this contract to the Ethereum network.

#### Alice (the contract user)
With Charlie's hard work finished, let's watch which ZoKrates commands Alice, the contract user, runs. Alice happens to know three numbers that sum to `15` but since numerical privacy is a big concern to her, she is especially attracted to Charlie's program. Alice chooses her public and private inputs accordingly:

    x = 5
    s1 = 3
    s2 = 7
 

She first computes her witness:

    $ ./target/release/zokrates compute-witness -a 5

*(ZoKrates is currently specified so that public arguments are passed following the `-a` flag but private arguments are passed interactively afterwards. Again ZoKrates is a WIP and may change in the future)*. Alice then passes ZoKrates her private values `3` and `7` after being prompted. Zokrates then creates a file called `witness` containing the public and private values making up the witness. Alice grab's the publically available `proving.key` file from Charlie and generates her proof.

    $ ./target/release/zokrates generate-proof

ZoKrates does not create a file for the proof, but prints the values to the console. It should look something like this:

    A = 0x4c764af0c2b189f8bec6237cfc4f2e12f93b16f874fffed45b14b144b8bd65f, 0x26743a72b82ca503ae3b3239cac1f946af5517fac0be790ea710f6c1bba4af29
	A_p = 0x1d54d989098411b2f0aca208bd200c7f1d6d75b52b884076b667e9748d8b37df, 0x25db886ec66bf13fdea009f36ce49ae595efbf0dc64ff0fb7403b30f68663337
	B = [0x20f17240a9bdd8541ca738ae97ba7e99f0b9b64cd401abda4587a2031d0ed98b, 0x241633137aaa2a8dd457818c0e82ae3e1837026c0013d2d68d015c77a01ec7b3], [0x116a698ca5d448b03bbdf54262315391e3c1fd8e5dd8fe2b77841b78a9dd14c9, 0x1fa0818cbc61dbf781daec1ddae207904c9d8228255d4b22232c4ba9886f62d2]
	B_p = 0x2c65f3f600576385fe95d260e9caa6311fb2fc687809a89c7113269564dd744e, 0x9ebacb7dce05042ba58daa9b3d7069006ac9ee6cbf5ad48894fcf8cfb6c58f8
	C = 0x2fb736ab7f86efc0207b161442af8bf73063b129215894024b15dafcc2bd83d0, 0xfc876ae710f329636f37d1892f0c9c7d8c0c062c07ef4aa28fa4a53e1c3940d
	C_p = 0x1ddfa7028e269d3d4a0fddb13c5f0900e5bbce9bb779ce54c445b9c5b4e91f61, 0x1c61e8a29861eeb37d9970133df2c5a51460cc06a3c7363ae1e85610e6ea34f3
	H = 0xeb9c23cbe785fc9416327bde3bc541821aa196899febbc7a0cb50cad9d6e01e, 0xa99068e9de82385d69499cf9d653bc1f7ff7cb0d88e83ae4c4d5f617de62210
	K = 0x131794e22bde4b0c1b44f4d2234fbce6b863e7b5088de9e8eaa6a3c57426e75b, 0x26f1a1b5fc9476817178e666011bde5731cdc52c8ce9b394f54bf518f16d3cb8

These are the eight variables that make up a zkSNARKs proof. Charlie's contract's `verifyTx` function works by accepting eight values like the ones above that represent a proof, along with an array of public inputs. The expected output of the program is also considered an public input, so Alice assembles her public input array as follows:

    input = [5, 1]

Then, in her preferred implementation of web3, she sends a state-changing transaction to this function. Since Alice did everything correctly, the `verifyTx` functions returns `true`. Its important to note that Alice's proof is hardwired to her witness. If she wanted to prove another set of values sum to `15` she would need to compute a new witness and a new proof from that witness. This new proof will also work with Charlie's existing deployed contract.

**Note:** *I have found that using .call() to access this function will throw an error when the program reaches a bit of assembly code I assume is making a call to the precompiled Byzantium contracts. I do not know why it is required to transact instead of call but that is the way it is. It is also important to set your gas limit high on this transaction as verification uses a lot of gas. If transactions are reverting try setting a high gas limit and work down from there.*



#### Bob (an observer)
Bob has been watching Charlie's contract closely.  As a skeptic, he does not want anyone to falsely claim they know three numbers that sum to `15` without proof. But Bob also respects provers' right to privacy and does not need to know all three numbers if a valid zero-knowledge proof is presented to him. He observes Alice's transaction, and since the `verifyTx` functions returns `true` he can remain ignorant of Alice's secret input while resting assured that Alice truly does know a set of three numbers that sum to `15`.


And that's how we can use ZoKrates to build, prove, and verify zero-knowledge proofs on the Ethereum blockchain! I hope you enjoyed this tutorial and found it useful to implement zkSNARKs yourself!

## Some final thoughts
### What sort of applications does zkSNARKS have?

zkSNARKs provides a layer of privacy for provers such as Alice. She can perform operations on values hidden from everyone else. But what if she wanted outside observers such as Bob to interact with her hidden values? It turns out it's not that simple. 

For example, when I started exploring zkSNARKs I thought it would be cool to build a "heads or tails" type guessing game on the blockchain that doesn't use a random oracle or a commit-reveal type scheme. Alice could commit a secret value (either `1` or `0`) to a proof stored inside a contract and Bob could then call a function submitting Alice's stored proof and his guess at the public argument to a `verifyTx` function. Something like:

    def main(guess):
	  secret == 1 || secret == 0
	  secret == guess
	  return 1

This would work, but the problem is that Alice has already made her proof public. Bob can test all two possibilities (`0` or `1`) offchain (since the proof and verification key are available to him) before ever submitting anything to the network, guaranteeing that every guess he makes to the mainnet will be correct, and thus always win. This could be expanded into a guessing game with more options, but the problem remains that it is still going to be easily brute-forceable offchain. 

Alternatively, Alice could store the hash of the answer to the blockchain, and observers like Bob could submit a proof that he knows the secret value, along with Alice's stored hash to a `verifyTX` function. Proofs could take the form:

    def main(hashOfAnswer):
	  hash(guess) == hashOfAnswer
	  return 1
	  
Unfortunately, in this example as well, it would be fairly easy for Bob to game the system since their are only two values, `0` and `1`. He can easily to compute their hashes offchain and see if they match the value stored in the contract before transacting. Alice could salt her hashes but this wouldn't make for a fun game either since now Bob's problem has changed to brute-forcing a reverse hash. 

It seems like applications involving zkSNARKs allow the prover to maintain a secret and perform operations on this secret themselves, but do not not allow anyone else to interact with this secret in a meaningful way. To outside observers, these secrets are either impossible to figure out and thus interact with (eg. reversing a salted hash), or trivially easy to brute-force offchain. 

In the Consensys article's example of a confidential transaction, it seemingly appears that two individuals are interacting over hidden values but in fact, the example is more akin to one individual sending tokens between two wallet addresses she owns herself. This is because both individuals need to know the secret `w.value` for the transaction to work. It's not prover Alice and observer Bob creating a transaction. It's Alice1 and Alice2 creating a confidential transaction.

Unfortunately, while this makes cool games like heads-or-tails impossible to implement, we can see that zkSNARKs are a powerful tool for maintaining individuals' privacy. They also work well in cases where two individuals share a secret value between themselves beforehand.

I hope you found this tutorial helpful! It helped me a lot to write my understanding down to feel like I understood how to use zkSNARKs in practice.


