# zkSNARKs: Driver's Ed.
## A practical beginner's guide to creating, proving, and verifying zkSNARKs in your contracts
Joseph Stockermans 
Jan. 6th 2018

![enter image description here](https://images.unsplash.com/photo-1499102443154-09b49b31a97b?auto=format&fit=crop&w=1500&q=80)

## Motivation
Like many people, when I first heard about the Byzantium update a few months ago, I was super excited for all the new privacy features that zkSNARKS would offer us - without really understanding anything about zkSNARKs at all. Nevertheless I was eager to write some contracts with secret values and stuff like that as soon as I got a chance. 

I recently took some time to explore the subject and my journey started with [this article from Consensys](https://media.consensys.net/introduction-to-zksnarks-with-examples-3283b554fc3b) written by Christian Lundkvist. It gives an excellent layman's explanation of zkSNARKs and drafts a token contract with hidden balances (if you have not read this article I highly recommend you do as I will refer to it a lot going forward). I thought this was very cool and decided to try to recreate this contract on my own. The contract code contains a mysterious function called `zksnarkverify`. I wasn't exactly sure what this was or what was included in the Byzantium update, so I initially assumed that this was some kind of new built-in solidity function to use zkSNARKs. Unfortunately, as it turns out, this function was just a vision of a function that *could* exist, and the actual implementation was left to the reader to create for themselves. 

I started Googling to find someone's implementation of this function and I was quickly overwhelmed by the difficulty of understanding most of the popular articles out there on the subject: Christian Reitwiessner's [zkSNARKs in a Nutshell](https://blog.ethereum.org/2016/12/05/zksnarks-in-a-nutshell/), Vitalik Buterin's 3-part series [Zk-SNARKs: Under the Hood](https://medium.com/@VitalikButerin/zk-snarks-under-the-hood-b33151a013f6), and this seemingly impenetrable [Solidity contract](https://gist.github.com/chriseth/f9be9d9391efc5beb9704255a8e2989d) that StackOverflow users were claiming to be ["the zksnarks solidity code"](https://ethereum.stackexchange.com/questions/16473/is-there-any-working-zk-snark-implementation-even-if-experimental-among-the-ex/21229). These all seemed very complicated to me and none of them really discussed getting up and running with zkSNARKs.  In fact many commenters on the "zksnarks contract" gist seemed to have no idea where to get started.

What I was looking for was a copy+paste from StackOverflow type answer. When I finally figured something out, I decided to write this tutorial for developers who just want to know how to drive the zkSNARKs car, and not how the car works - so I hope you find this tutorial helpful!



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

Great! Now we're inside a Docker container that has all the ZoKrates commands available to us. Before we run anything, I'm going to provide a little description for each of the ZoKrates commands found on their GitHub README, since there is currently no explanation for them.

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
Meanwhile, Bob has been watching Charlie's contract. He sees that Alice has sent a transaction to the `verifyTx` function, and that this function returned `true`. He can be certain that this function will only output `true` if Alice provided a proof generated from Charlie's proving-key `pk` and if Alice's public and private input fulfill the necessary constraints of Charlie's program. In other words, he can be sure Alice's proof is valid while still knowing nothing about her private inputs into the proof.

## Taking zkSNARKs for a Test Drive
### A concrete example
#### Charlie (a trusted third party)
First let's watch which ZoKrates commands Charlie, the trusted party, runs. Charlie thinks that it would be beneficial to the Etheruem community to have a contract in which users can prove that they know a set of three numbers that sum to `15`. He believes that this type of proof would benefit from some privacy so he designs a program that accepts one public and two private inputs and asserts that the sum of the three is equal to `15`.

Charlie's program looks something like this, and assuming we are in the ZoKrates root directory, he saves it to a file called 
`sumsToFifteen.code`.

    def main(x, private s1, private s2):
	    s1 + s2 + x == 15
	    return 1

In Charlie's program, `x` is the public input, and `s1` and `s2` are secret values *(ZoKrates is currently specified so that secret values are explicitly passed to the function, with a `private` keyword. Bear in mind that ZoKrates is a new project and a WIP so many details may be subject to change in the future)*. The second line of the program is an assertion statement. Users will not be able to generate proofs for this program if values passed to it cause the the assertion to fail. On the third line, we see that if a proof constructed from this program is verified, it will output `1`.

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

    $ ./target/release/zokrates compute-witness -a 5 3 7

*(ZoKrates is currently specified so that all arguments are passed following the `-a` flag. Again ZoKrates is a WIP and may change in the future)*. Zokrates then creates a file called `witness` containing the public and private values making up the witness. Alice grabs the publically available `proving.key` file from Charlie and generates her proof.

    $ ./target/release/zokrates generate-proof

ZoKrates does not create a file for the proof, but prints the values to the console. It should look something like this:

    A = 0x1628f3170cc16d40aad2e8fa1ab084f542fcb12e75ce1add62891dd75ba1ffd7, 0x11b20d11a0da724e41f7e2dc4d217b3f068b4e767f521a9ea371e77e496cc54
    A_p = 0x1a4406c4ab38715a6f7624ece480aa0e8ca0413514d70506856af0595a853bc3, 0x2553e174040723a6bf5ea2188d2a1429bb01b13084c4af5b51701e6077716980
    B = [0x27c9878700f09edc60cf23d3fb486fe50726f136ff46ad48653a3e7254ae3020, 0xe35b33188dc2f47618248e4f12a97026c3acdef9b4d021bf94e7b6d9e8ffbb6], [0x64cf25d53d57e2931d58d22fe34122fa12def64579c02d0227a496f31678cf8, 0x26212d004463c9ff80fc65f1f32321333b90de63b6b35805ef24be8b692afb28]
    B_p = 0x175e0abe73317b738fd5e9fd1d2e3cb48124be9f7ae8080b8dbe419b224e96a6, 0x85444b7ef6feafa8754bdd3ca0be17d245f13e8cc89c37e7451b55555f6ce9d
    C = 0x297a60f02d72bacf12a58bae75d4f330bed184854c3171adc6a65bb708466a76, 0x16b72260e7854535b0a821dd41683a28c89b0d9fcd77d36a157ba709996b490
    C_p = 0x29ea33c3da75cd937e86aaf6503ec67d18bde775440da90a492966b2eb9081fe, 0x13fcc4b019b05bc82cd95a6c8dc880d4da92c53abd2ed449bd393e5561d21583
    H = 0x2693e070bade67fb06a55fe834313f97e3562aa42c46d33c73fccb8f9fd9c2de, 0x26415689c4f4681680201c1975239c8f454ac4b2217486bc26d92e9dcacb58d7
    K = 0x11afe3c25ff3821b8b42fde5a85b734cf6000c4b77ec57e08ff5d4386c60c72a, 0x24174487b1d642e4db86689542b8d6d9e97ec56fcd654051e96e36a8b74ea9ef

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

## Appendix 
### Example Solidity Contracts (Charlie)

#### SumsToFifteen.sol
##### (Custom contract)

    pragma solidity ^0.4.14;
    import './Verifier.sol';
    contract SumsToFifteen is Verifier {
        bool public success = false;
        function verifyFifteen(
            uint[2] a,
            uint[2] a_p,
            uint[2][2] b,
            uint[2] b_p,
            uint[2] c,
            uint[2] c_p,
            uint[2] h,
            uint[2] k,
            uint[2] input) public {
            // Verifiy the proof
            success = verifyTx(a, a_p, b, b_p, c, c_p, h, k, input);
            if (success) {
                // Proof verified
            } else {
                // Sorry, bad proof!
            }
        }
    }

 

#### Verifier.sol 
##### (Output from ZoKrates `export-verifier` command)

    pragma solidity ^0.4.14;
    library Pairing {
        struct G1Point {
            uint X;
            uint Y;
        }
        // Encoding of field elements is: X[0] * z + X[1]
        struct G2Point {
            uint[2] X;
            uint[2] Y;
        }
        /// @return the generator of G1
        function P1() internal returns (G1Point) {
            return G1Point(1, 2);
        }
        /// @return the generator of G2
        function P2() internal returns (G2Point) {
            return G2Point(
                [11559732032986387107991004021392285783925812861821192530917403151452391805634,
                 10857046999023057135944570762232829481370756359578518086990519993285655852781],
                [4082367875863433681332203403145435568316851327593401208105741076214120093531,
                 8495653923123431417604973247489272438418190587263600148770280649306958101930]
            );
        }
        /// @return the negation of p, i.e. p.add(p.negate()) should be zero.
        function negate(G1Point p) internal returns (G1Point) {
            // The prime q in the base field F_q for G1
            uint q = 21888242871839275222246405745257275088696311157297823662689037894645226208583;
            if (p.X == 0 && p.Y == 0)
                return G1Point(0, 0);
            return G1Point(p.X, q - (p.Y % q));
        }
        /// @return the sum of two points of G1
        function add(G1Point p1, G1Point p2) internal returns (G1Point r) {
            uint[4] memory input;
            input[0] = p1.X;
            input[1] = p1.Y;
            input[2] = p2.X;
            input[3] = p2.Y;
            bool success;
            assembly {
                success := call(sub(gas, 2000), 6, 0, input, 0xc0, r, 0x60)
                // Use "invalid" to make gas estimation work
                switch success case 0 { invalid }
            }
            require(success);
        }
        /// @return the product of a point on G1 and a scalar, i.e.
        /// p == p.mul(1) and p.add(p) == p.mul(2) for all points p.
        function mul(G1Point p, uint s) internal returns (G1Point r) {
            uint[3] memory input;
            input[0] = p.X;
            input[1] = p.Y;
            input[2] = s;
            bool success;
            assembly {
                success := call(sub(gas, 2000), 7, 0, input, 0x80, r, 0x60)
                // Use "invalid" to make gas estimation work
                switch success case 0 { invalid }
            }
            require (success);
        }
        /// @return the result of computing the pairing check
        /// e(p1[0], p2[0]) *  .... * e(p1[n], p2[n]) == 1
        /// For example pairing([P1(), P1().negate()], [P2(), P2()]) should
        /// return true.
        function pairing(G1Point[] p1, G2Point[] p2) internal returns (bool) {
            require(p1.length == p2.length);
            uint elements = p1.length;
            uint inputSize = elements * 6;
            uint[] memory input = new uint[](inputSize);
            for (uint i = 0; i < elements; i++)
            {
                input[i * 6 + 0] = p1[i].X;
                input[i * 6 + 1] = p1[i].Y;
                input[i * 6 + 2] = p2[i].X[0];
                input[i * 6 + 3] = p2[i].X[1];
                input[i * 6 + 4] = p2[i].Y[0];
                input[i * 6 + 5] = p2[i].Y[1];
            }
            uint[1] memory out;
            bool success;
            assembly {
                success := call(sub(gas, 2000), 8, 0, add(input, 0x20), mul(inputSize, 0x20), out, 0x20)
                // Use "invalid" to make gas estimation work
                switch success case 0 { invalid }
            }
            require(success);
            return out[0] != 0;
        }
        /// Convenience method for a pairing check for two pairs.
        function pairingProd2(G1Point a1, G2Point a2, G1Point b1, G2Point b2) internal returns (bool) {
            G1Point[] memory p1 = new G1Point[](2);
            G2Point[] memory p2 = new G2Point[](2);
            p1[0] = a1;
            p1[1] = b1;
            p2[0] = a2;
            p2[1] = b2;
            return pairing(p1, p2);
        }
        /// Convenience method for a pairing check for three pairs.
        function pairingProd3(
                G1Point a1, G2Point a2,
                G1Point b1, G2Point b2,
                G1Point c1, G2Point c2
        ) internal returns (bool) {
            G1Point[] memory p1 = new G1Point[](3);
            G2Point[] memory p2 = new G2Point[](3);
            p1[0] = a1;
            p1[1] = b1;
            p1[2] = c1;
            p2[0] = a2;
            p2[1] = b2;
            p2[2] = c2;
            return pairing(p1, p2);
        }
        /// Convenience method for a pairing check for four pairs.
        function pairingProd4(
                G1Point a1, G2Point a2,
                G1Point b1, G2Point b2,
                G1Point c1, G2Point c2,
                G1Point d1, G2Point d2
        ) internal returns (bool) {
            G1Point[] memory p1 = new G1Point[](4);
            G2Point[] memory p2 = new G2Point[](4);
            p1[0] = a1;
            p1[1] = b1;
            p1[2] = c1;
            p1[3] = d1;
            p2[0] = a2;
            p2[1] = b2;
            p2[2] = c2;
            p2[3] = d2;
            return pairing(p1, p2);
        }
    }
    contract Verifier {
        using Pairing for *;
        struct VerifyingKey {
            Pairing.G2Point A;
            Pairing.G1Point B;
            Pairing.G2Point C;
            Pairing.G2Point gamma;
            Pairing.G1Point gammaBeta1;
            Pairing.G2Point gammaBeta2;
            Pairing.G2Point Z;
            Pairing.G1Point[] IC;
        }
        struct Proof {
            Pairing.G1Point A;
            Pairing.G1Point A_p;
            Pairing.G2Point B;
            Pairing.G1Point B_p;
            Pairing.G1Point C;
            Pairing.G1Point C_p;
            Pairing.G1Point K;
            Pairing.G1Point H;
        }
        function verifyingKey() internal returns (VerifyingKey vk) {
            vk.A = Pairing.G2Point([0x11cdfdd85c8506e01b1013980776315a6d861d5505fe3b2d70ca66646f08adea, 0x1d831f34d31e8094f09b7fd8249f545073ef3f8d1038bf71248529a143fdebf3], [0x17ec9e42908c8624c32dcdb7b76c64acf53b6d4ee72444d3a7c1fe8e00f5d7fa, 0x2fbea9b20d6443c407819f819e0ade7a07c030e9f62894dbf92dad1a55a1614d]);
            vk.B = Pairing.G1Point(0x10cf27740851709989f00203df5525ecc3577bf044eadef05037453b95c35f40, 0x224686b18747c89d9bef1039e945efb497ee7ff669011c14c6ca3a7778578324);
            vk.C = Pairing.G2Point([0xd02131a4f3d1f61e40b25666540d93b7db79f76fa35f0f9d7a75ac2a4e23856, 0x4fb6e315e432313cdb9fbce5660354dd6fb56432b7efa72f8bf9c38cd750ef1], [0x1ae71edad4c83cda00b5a545e3f530f15fff83863d2db70b47f1a640c3c4f154, 0x224004ca2456ac33a7f1c9594b9b30f31370f432907ac4b5e72664b74254b9a7]);
            vk.gamma = Pairing.G2Point([0x16375cb516444080a3d506e3ccda9f36a9b533d32732600125fe23b95d424719, 0x14b036bf21e18810e6b3549f868b9897c1bc0a5a50b17a3b2a0db67215795254], [0x261fbb7c90d8f47bb64b866d8be09b66d5b074f1ca7329310473dc7eaf167eb3, 0x2c05775885afcb722cd4d7c832a6d00b08d108697b044d2ac05ab595dfcf6f1a]);
            vk.gammaBeta1 = Pairing.G1Point(0x1b86641088fe2985bd3c4b1195e7792ae61314561a20bd28622a3b10716d5b79, 0x272f35e571a9c28f54c969a0e093f3e2d92962857f67bdf6122e98b14389153);
            vk.gammaBeta2 = Pairing.G2Point([0x961ab4f5fd50c437b4aace5efda67eb910235662a3e1d4f4c07073361ad8c14, 0x16585be5bc76e6fb26750d0361efc293ff7ce6fe63d40efa936dcda52e89d88a], [0x12d8cc318710026ebdd483d4a8e26951e7728682195488aa8231d33ce55a7211, 0x1babe275048dbabbda7200de77538ba582abc0857030ddeef52ea64c31406d8a]);
            vk.Z = Pairing.G2Point([0x1532128acd3ae189f8cac212620f6e93a47da24350534a5383804e18b7ec4497, 0x2014cf88774f460a6f003dd1a4d6e40db890de1ca4694219ea1245041f73be03], [0xd368d63297b3ee0e42219e1b938555a3d7d2249937bc683f195043803590583, 0x1278c675d5eecfde121e1d95ac0b1f8c8a6fa920f176f016893acebecdf478e5]);
            vk.IC = new Pairing.G1Point[](3);
            vk.IC[0] = Pairing.G1Point(0x10682c32b1cbdd9d48d08ba6397853ae73ee82dfc2441dd345922460cc78d508, 0x2fc117c0cdb41a93bdf87416122359c6d065f689c02f08d9d86924e8d0328d77);
            vk.IC[1] = Pairing.G1Point(0x2188a0de08297f728150224fe8a0796a17d7499d5f774b9b765b324cfbbb2ea9, 0x2ceb95f152b45bd36d8662e0d211d1b3fc912fe6935ec400d7e2fb5d7c27e3da);
            vk.IC[2] = Pairing.G1Point(0x13c07e1ab1bd0dc50a121c54b1774d758c9eebb83673f728678a92528253832c, 0xe38e2d5cfc5c4e6b4f63e5e4e1496b86fd8b627c78af3e130cda3ad5c6f328d);
        }
        function verify(uint[] input, Proof proof) internal returns (uint) {
            VerifyingKey memory vk = verifyingKey();
            require(input.length + 1 == vk.IC.length);
            // Compute the linear combination vk_x
            Pairing.G1Point memory vk_x = Pairing.G1Point(0, 0);
            for (uint i = 0; i < input.length; i++)
                vk_x = Pairing.add(vk_x, Pairing.mul(vk.IC[i + 1], input[i]));
            vk_x = Pairing.add(vk_x, vk.IC[0]);
            if (!Pairing.pairingProd2(proof.A, vk.A, Pairing.negate(proof.A_p), Pairing.P2())) return 1;
            if (!Pairing.pairingProd2(vk.B, proof.B, Pairing.negate(proof.B_p), Pairing.P2())) return 2;
            if (!Pairing.pairingProd2(proof.C, vk.C, Pairing.negate(proof.C_p), Pairing.P2())) return 3;
            if (!Pairing.pairingProd3(
                proof.K, vk.gamma,
                Pairing.negate(Pairing.add(vk_x, Pairing.add(proof.A, proof.C))), vk.gammaBeta2,
                Pairing.negate(vk.gammaBeta1), proof.B
            )) return 4;
            if (!Pairing.pairingProd3(
                    Pairing.add(vk_x, proof.A), proof.B,
                    Pairing.negate(proof.H), vk.Z,
                    Pairing.negate(proof.C), Pairing.P2()
            )) return 5;
            return 0;
        }
        event Verified(string);
        function verifyTx(
                uint[2] a,
                uint[2] a_p,
                uint[2][2] b,
                uint[2] b_p,
                uint[2] c,
                uint[2] c_p,
                uint[2] h,
                uint[2] k,
                uint[2] input
            ) returns (bool r) {
            Proof memory proof;
            proof.A = Pairing.G1Point(a[0], a[1]);
            proof.A_p = Pairing.G1Point(a_p[0], a_p[1]);
            proof.B = Pairing.G2Point([b[0][0], b[0][1]], [b[1][0], b[1][1]]);
            proof.B_p = Pairing.G1Point(b_p[0], b_p[1]);
            proof.C = Pairing.G1Point(c[0], c[1]);
            proof.C_p = Pairing.G1Point(c_p[0], c_p[1]);
            proof.H = Pairing.G1Point(h[0], h[1]);
            proof.K = Pairing.G1Point(k[0], k[1]);
            uint[] memory inputValues = new uint[](input.length);
            for(uint i = 0; i < input.length; i++){
                inputValues[i] = input[i];
            }
            if (verify(inputValues, proof) == 0) {
                Verified("Transaction successfully verified.");
                return true;
            } else {
                return false;
            }
        }
    }

### Example Web3 in Python (Alice)

    from web3.contract import ConciseContract
    from web3 import Web3, HTTPProvider
    import json
    
    web3 = Web3(HTTPProvider('http://localhost:8545'))
    
    # Get the address and ABI of the contract
    address = '0xB4D425cF6fFa2ACD4d8e08A5427F846FCF20899f'
    abi = json.loads('[{"constant": true, "inputs": [], "name": "success", "outputs": [{"name": "", "type": "bool"}], "payable": false, "stateMutability": "view", "type": "function"}, {"constant": false, "inputs": [{"name": "a", "type": "uint256[2]"}, {"name": "a_p", "type": "uint256[2]"}, {"name": "b", "type": "uint256[2][2]"}, {"name": "b_p", "type": "uint256[2]"}, {"name": "c", "type": "uint256[2]"}, {"name": "c_p", "type": "uint256[2]"}, {"name": "h", "type": "uint256[2]"}, {"name": "k", "type": "uint256[2]"}, {"name": "input", "type": "uint256[2]"}], "name": "verifyFifteen", "outputs": [], "payable": false, "stateMutability": "nonpayable", "type": "function"}, {"constant": true, "inputs": [], "name": "flag", "outputs": [{"name": "", "type": "uint256"}], "payable": false, "stateMutability": "view", "type": "function"}, {"constant": false, "inputs": [{"name": "a", "type": "uint256[2]"}, {"name": "a_p", "type": "uint256[2]"}, {"name": "b", "type": "uint256[2][2]"}, {"name": "b_p", "type": "uint256[2]"}, {"name": "c", "type": "uint256[2]"}, {"name": "c_p", "type": "uint256[2]"}, {"name": "h", "type": "uint256[2]"}, {"name": "k", "type": "uint256[2]"}, {"name": "input", "type": "uint256[2]"}], "name": "verifyTx", "outputs": [{"name": "r", "type": "bool"}], "payable": false, "stateMutability": "nonpayable", "type": "function"}, {"anonymous": false, "inputs": [{"indexed": false, "name": "", "type": "string"}], "name": "Verified", "type": "event"}]')
    
    # Create a contract object
    contract = web3.eth.contract(
        address, 
        abi=abi,
        ContractFactoryClass=ConciseContract,
    )
    
    # Proof generated from Zokrates
    A = [0x1628f3170cc16d40aad2e8fa1ab084f542fcb12e75ce1add62891dd75ba1ffd7, 0x11b20d11a0da724e41f7e2dc4d217b3f068b4e767f521a9ea371e77e496cc54]
    A_p = [0x1a4406c4ab38715a6f7624ece480aa0e8ca0413514d70506856af0595a853bc3, 0x2553e174040723a6bf5ea2188d2a1429bb01b13084c4af5b51701e6077716980]
    B = [[0x27c9878700f09edc60cf23d3fb486fe50726f136ff46ad48653a3e7254ae3020, 0xe35b33188dc2f47618248e4f12a97026c3acdef9b4d021bf94e7b6d9e8ffbb6], [0x64cf25d53d57e2931d58d22fe34122fa12def64579c02d0227a496f31678cf8, 0x26212d004463c9ff80fc65f1f32321333b90de63b6b35805ef24be8b692afb28]]
    B_p = [0x175e0abe73317b738fd5e9fd1d2e3cb48124be9f7ae8080b8dbe419b224e96a6, 0x85444b7ef6feafa8754bdd3ca0be17d245f13e8cc89c37e7451b55555f6ce9d]
    C = [0x297a60f02d72bacf12a58bae75d4f330bed184854c3171adc6a65bb708466a76, 0x16b72260e7854535b0a821dd41683a28c89b0d9fcd77d36a157ba709996b490]
    C_p = [0x29ea33c3da75cd937e86aaf6503ec67d18bde775440da90a492966b2eb9081fe, 0x13fcc4b019b05bc82cd95a6c8dc880d4da92c53abd2ed449bd393e5561d21583]
    H = [0x2693e070bade67fb06a55fe834313f97e3562aa42c46d33c73fccb8f9fd9c2de, 0x26415689c4f4681680201c1975239c8f454ac4b2217486bc26d92e9dcacb58d7]
    K = [0x11afe3c25ff3821b8b42fde5a85b734cf6000c4b77ec57e08ff5d4386c60c72a, 0x24174487b1d642e4db86689542b8d6d9e97ec56fcd654051e96e36a8b74ea9ef]
    
    # Set gas and gas price
    params = {
        'gasPrice':20000000000, 
        'gas': 4000000, 
        'from':web3.eth.accounts[0],
    }
    
    # Correct input
    I = [5, 1]
    
    # Verify
    txhash = contract.verifyFifteen(A, A_p, B, B_p, C, C_p, H, K, I, transact=params)
    
    # Check success
    success = contract.success()
    print(success)
    >> True
    
    # Incorrect input
    I = [6, 1]
    
    # Verify
    txhash = contract.verifyFifteen(A, A_p, B, B_p, C, C_p, H, K, I, transact=params)
    
    # Check success
    success = contract.success()
    print(success)
    >> False
