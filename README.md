 # circom + snarkjs + WebAssembly +  Groth16 zk-SNARK protocol

 a demo project to demonstrate Groth16 based proof system. circom is used to define constraints that define the arithmetic circuit.  
 
 Computes the witness with WebAssembly.  

 Use the `snarkjs` tool to generate and validate a proof for the inputs. `snarkjs` used to conduct "powers of tau" ceremony for creating the trusted setup and also provide the  commands to create and verify Groth16 proofs.  

 ##Install dependencies  

 Install necessary tools

 ### Rust
 `curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh`
 ### Node & npm
 `sudo apt install nodejs`  
 `sudo apt install npm`  
 ### Circom
 To install from our sources, clone the circom repository:  

 `git clone https://github.com/iden3/circom.git`  

 Enter the circom directory and use the cargo build to compile:  
 
 `cargo build --release`  

 The installation takes around 3 minutes to be completed. When the command successfully finishes, it generates the circom binary in the directory target/release. You can install this binary as follows:
 
 `cargo install --path circom`  

 The previous command will install the circom binary in the directory $HOME/.cargo/bin.  
 
 Now, you should be able to see all the options of the executable by using the help flag:  
 
 circom --help  
 
    Circom Compiler 2.0.0  
    IDEN3  
    Compiler for the Circom programming language  
   
    USAGE:  
       circom [FLAGS] [OPTIONS] [input]  
 
    FLAGS:  
       -h, --help       Prints help information  
          --inspect    Does an additional check over the constraints produced  
          --O0         No simplification is applied  
       -c, --c          Compiles the circuit to c  
          --json       outputs the constraints in json format  
          --r1cs       outputs the constraints in r1cs format  
          --sym        outputs witness in sym format  
          --wasm       Compiles the circuit to wasm  
          --wat        Compiles the circuit to wat  
          --O1         Only applies var to var and var to constant simplification  
       -V, --version    Prints version information  
 
    OPTIONS:  
          --O2 <full_simplification>    Full constraint simplification [default: full]  
       -o, --output <output>             Path to the directory where the output will be written [default: .]  
 
    ARGS:  
       <input>    Path to a circuit with a main component [default: ./circuit.circom]  
 ### snarkjs
 snarkjs is a npm package that contains code to generate and validate ZK proofs from the artifacts produced by circom.  
 
 You can install snarkjs with the following command:  
 
 `npm install -g snarkjs`  

 ## Uses  
 Write following code and save into a file named `multiplier2.circom`  

 ```pragma circom 2.0.0;  

 template Multiplier2() {
    signal input a;
    signal input b;
    signal output c;
    c <== a*b;
 }  

 component main = Multiplier2();  
 ```
 ## Compiling circuit  

 `circom multiplier2.circom --r1cs --wasm --sym --c`  
 With these options we generate three types of files:  
 
 `--r1cs`: it generates the file multiplier2.r1cs that contains the R1CS constraint system of the circuit in binary format.  
 `--wasm`: it generates the directory multiplier2_js that contains the Wasm code (multiplier2.wasm) and other files needed to generate the witness.  
 `--sym` : it generates the file multiplier2.sym , a symbols file required for debugging or for printing the constraint system in an annotated mode.  
 `--c` : it generates the directory multiplier2_cpp that contains several files (multiplier2.cpp, multiplier2.dat, and other common files for every compiled program like main.cpp, MakeFile, etc) needed to compile the C code to generate the witness.  
 We can use the option `-o` to specify the directory where these files are created.  
 
 Since version 2.0.8, we can use the option -l to indicate the directory where the directive include should look for the circuits indicated.  

 ## prepare the input file
 Save following data into a file called `input.json`  

 `{"a": "3", "b": "11"}`  
 We will prove that we know two numers a and b such that their multiplication  is 33 without revealing a and b.  

 ## Computing the witness with WebAssembly  
 Enter in the directory multiplier2_js, add the input in a file input.json and execute:

 `node generate_witness.js multiplier2.wasm input.json witness.wtns`

 ## Proving circuits
After compiling the circuit and running the witness calculator with an appropriate input, we will have a file with extension .wtns that contains all the computed signals and, a file with extension .r1cs that contains the constraints describing the circuit. Both files will be used to create our proof.

Now, we will use the snarkjs tool to generate and validate a proof for our input. In particular, using the multiplier2, we will prove that we are able to provide the two factors of the number 33. That is, we will show that we know two integers a and b such that when we multiply them, it results in the number 33.

We are going to use the Groth16 zk-SNARK protocol. To use this protocol, you will need to generate a trusted setup. Groth16 requires a per circuit trusted setup. In more detail, the trusted setup consists of 2 parts:

The powers of tau, which is independent of the circuit.
The phase 2, which depends on the circuit.
Next, we provide a very basic ceremony for creating the trusted setup and we also provide the basic commands to create and verify Groth16 proofs. Review the related Background section and check the snarkjs tutorial for further information.

Powers of Tau
First, we start a new "powers of tau" ceremony:

snarkjs powersoftau new bn128 12 pot12_0000.ptau -v
Then, we contribute to the ceremony:

snarkjs powersoftau contribute pot12_0000.ptau pot12_0001.ptau --name="First contribution" -v
Now, we have the contributions to the powers of tau in the file pot12_0001.ptau and we can proceed with the Phase 2.

Phase 2
The phase 2 is circuit-specific. Execute the following command to start the generation of this phase:

snarkjs powersoftau prepare phase2 pot12_0001.ptau pot12_final.ptau -v
Next, we generate a .zkey file that will contain the proving and verification keys together with all phase 2 contributions. Execute the following command to start a new zkey:

snarkjs groth16 setup multiplier2.r1cs pot12_final.ptau multiplier2_0000.zkey
Contribute to the phase 2 of the ceremony:

snarkjs zkey contribute multiplier2_0000.zkey multiplier2_0001.zkey --name="1st Contributor Name" -v
Export the verification key:

snarkjs zkey export verificationkey multiplier2_0001.zkey verification_key.json
Generating a Proof
Once the witness is computed and the trusted setup is already executed, we can generate a zk-proof associated to the circuit and the witness:

snarkjs groth16 prove multiplier2_0001.zkey witness.wtns proof.json public.json
This command generates a Groth16 proof and outputs two files:

proof.json: it contains the proof.
public.json: it contains the values of the public inputs and outputs.
Verifying a Proof
To verify the proof, execute the following command:

snarkjs groth16 verify verification_key.json public.json proof.json
The command uses the files verification_key.json we exported earlier,proof.json and public.json to check if the proof is valid. If the proof is valid, the command outputs an OK.

A valid proof not only proves that we know a set of signals that satisfy the circuit, but also that the public inputs and outputs that we use match the ones described in the public.json file.

Verifying from a Smart Contract
​👉 It is also possible to generate a Solidity verifier that allows verifying proofs on Ethereum blockchain.

First, we need to generate the Solidity code using the command:

snarkjs zkey export solidityverifier multiplier2_0001.zkey verifier.sol
This command takes validation key multiplier2_0001.zkey and outputs Solidity code in a file named verifier.sol. You can take the code from this file and cut and paste it in Remix. You will see that the code contains two contracts: Pairing and Verifier. You only need to deploy the Verifier contract.

You may want to use first a testnet like Rinkeby, Kovan or Ropsten. You can also use the JavaScript VM, but in some browsers the verification takes long and the page may freeze.

The Verifier has a view function called verifyProof that returns TRUE if and only if the proof and the inputs are valid. To facilitate the call, you can use snarkJS to generate the parameters of the call by typing:

snarkjs generatecall
Cut and paste the output of the command to the parameters field of the verifyProof method in Remix. If everything works fine, this method should return TRUE. You can try to change just a single bit of the parameters, and you will see that the result is verifiable FALSE.




 
 
 