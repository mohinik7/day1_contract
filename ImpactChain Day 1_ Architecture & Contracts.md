

# **ImpactChain Hackathon: Day 1 Implementation Guide**

## **Engineering the Foundation for Verifiable Impact**

The primary objective for Day 1 is to construct the foundational architecture of the ImpactChain smart contract system. This initial phase is the most critical stage of the development sprint. It is here that the core promise of the platform—creating cryptographic truth in philanthropy—is forged into immutable on-chain logic.1 Success on Day 1 is not measured merely by lines of code written, but by the establishment of a robust, secure, and scalable foundation upon which all subsequent features will be built.

The key deliverables for this phase are a fully configured development environment connected to the Polygon Mumbai testnet, a meticulously designed two-contract architecture that ensures security and modularity, the core data structures and functions for on-chain project creation and donation, and a comprehensive suite of unit tests to validate this foundational logic.

Development will be guided by the project's core principles of "Security First" and "Scalability & Low Cost".1 The technical decisions outlined in this guide—from the choice of development framework to the specific smart contract design patterns—are made to directly serve these principles, ensuring the resulting prototype is not only functional but also architecturally sound and aligned with the project's long-term vision.

## **Section 1: Environment Configuration for the Polygon Mumbai Testnet**

A professional, secure, and efficient development environment is paramount to mitigating common setup delays that can derail a time-constrained hackathon project. This section provides a detailed walkthrough for establishing the complete toolchain required for ImpactChain development.

### **1.1 Initializing the Hardhat Development Environment**

Hardhat is a professional Ethereum development environment that facilitates compiling, deploying, testing, and debugging smart contracts.2 Its selection is a strategic decision to maximize development velocity. The maturity of the Hardhat ecosystem, particularly its integration with the Polygon network, provides a well-documented and stable foundation, significantly reducing the risk of time-consuming configuration and debugging issues that can arise with less established tooling.4

To begin, create a dedicated project directory and initialize a new Node.js project by executing the following commands in a terminal:

Bash

mkdir impactchain-hackathon  
cd impactchain-hackathon  
npm init \-y

Next, install Hardhat as a development dependency:

Bash

npm install \--save-dev hardhat

With Hardhat installed, run the project setup wizard:

Bash

npx hardhat

When prompted, select the Create a JavaScript project option. This will scaffold a standard project structure, including contracts, scripts, and test directories, along with a default hardhat.config.js file, providing a clean and organized starting point.2

### **1.2 Installing and Configuring Core Dependencies**

The following dependencies are essential for building, testing, and securely managing the ImpactChain smart contracts. Install them in a single command:

Bash

npm install \--save-dev @nomicfoundation/hardhat-toolbox @openzeppelin/contracts dotenv

Each of these packages serves a specific, critical role:

* **@nomicfoundation/hardhat-toolbox**: This is a modern, all-in-one package that bundles essential plugins and libraries, including ethers.js for blockchain interaction and testing frameworks like Mocha and Chai, simplifying the setup process.2  
* **@openzeppelin/contracts**: This library is the industry standard for secure, community-vetted smart contract components. It directly fulfills the "Security First" principle by providing battle-tested implementations of common standards and access control patterns, such as Ownable.sol and ReentrancyGuard.sol, which will be used to mitigate common vulnerabilities.1  
* **dotenv**: This is a critical utility for managing environment variables. It allows sensitive information, such as wallet private keys and API keys, to be loaded from a local .env file, preventing them from being accidentally hard-coded and committed to version control—a major security risk.2

### **1.3 Crafting the hardhat.config.js File for Polygon Connectivity**

The hardhat.config.js file is the central configuration hub for the project. It defines the Solidity compiler version, network connections, and other project settings. Replace the contents of the default file with the following configuration, which is specifically tailored for deployment to the Polygon Mumbai testnet.2

JavaScript

require("@nomicfoundation/hardhat-toolbox");  
require("dotenv").config();

/\*\* @type import('hardhat/config').HardhatUserConfig \*/  
module.exports \= {  
  solidity: "0.8.24", // Specifies the Solidity compiler version.  
  networks: {  
    mumbai: {  
      // Defines the configuration for the Polygon Mumbai testnet.  
      url: process.env.POLYGON\_MUMBAI\_RPC\_URL, // RPC endpoint URL loaded from the.env file.  
      accounts:, // Deployer wallet private key loaded from.env.  
    },  
  },  
};

This configuration performs two key actions:

1. It imports the dotenv package to enable the loading of environment variables.  
2. It defines a network named mumbai, configuring it to use the RPC URL and private key stored in the .env file. This allows for seamless deployment and interaction with the Polygon testnet via simple Hardhat commands (e.g., npx hardhat run scripts/deploy.js \--network mumbai).

### **1.4 Secure Credential Management with dotenv**

To protect sensitive credentials, create a file named .env in the root directory of the project. This file will store the environment variables referenced in hardhat.config.js. It is crucial to also create a .gitignore file and add .env to it to prevent this file from ever being tracked by Git.

Create the .env file and populate it with the following variables:

| Variable Name | Description | Example Value |
| :---- | :---- | :---- |
| POLYGON\_MUMBAI\_RPC\_URL | The JSON-RPC endpoint URL for the Polygon Mumbai testnet, obtained from a node provider like Alchemy or Infura. | https://polygon-mumbai.g.alchemy.com/v2/YOUR\_API\_KEY |
| DEPLOYER\_PRIVATE\_KEY | The private key of the wallet address that will be used to deploy the contracts and act as the initial owner/verifier. **Must be prefixed with 0x**. | 0xabcdef123456... |

To obtain a POLYGON\_MUMBAI\_RPC\_URL, sign up for a free account with a node provider service such as Alchemy or Infura. Create a new application, select the Polygon network and the Mumbai testnet, and copy the provided HTTP endpoint URL.4

### **1.5 Deployer Wallet Preparation: Generation and Funding**

For the hackathon, it is best practice to generate a new, dedicated wallet to serve as the deployer. This can be done easily using a browser extension wallet like MetaMask. Creating a new wallet isolates the hackathon activities and funds from any personal assets.

Once the wallet is created, its address must be funded with testnet MATIC to cover the gas fees for contract deployment and transactions. Testnet MATIC has no real-world value and can be obtained for free from services known as "faucets." Several reliable faucets are available for the Polygon Mumbai network 10:

* **Official Polygon Faucet**: [https://faucet.polygon.technology/](https://faucet.polygon.technology/) 12  
* **QuickNode Faucet**: [https://faucet.quicknode.com/polygon/mumbai](https://faucet.quicknode.com/polygon/mumbai) 10  
* **Alchemy Faucet**: [https://www.alchemy.com/faucets/polygon-mumbai](https://www.alchemy.com/faucets/polygon-mumbai) 13

Visit one of these sites, connect the newly created wallet or paste its public address, and request the testnet tokens. The funds may take a few minutes to appear in the wallet.11 After funding, export the wallet's private key from MetaMask and add it to the

.env file as the value for DEPLOYER\_PRIVATE\_KEY.

## **Section 2: Architecting the On-Chain Logic: A Two-Contract System**

The on-chain architecture is designed for security, clarity, and extensibility. It employs a two-contract system to ensure a clear separation of concerns, a fundamental principle of robust software design.1 This approach not only simplifies development and auditing but also lays a future-proof foundation for the platform's evolution.

### **2.1 The Principle of Separated Concerns: $ProjectEscrow and $ImpactToken**

The system's on-chain logic is divided between two specialized smart contracts:

1. **$ProjectEscrow.sol**: This contract is the core engine of the platform. It manages the entire lifecycle of a philanthropic project, including its creation, donation handling, milestone tracking, verification events, and the conditional release of funds. It encapsulates all the complex business logic and state transitions.1  
2. **$ImpactToken.sol**: This is a standard ERC-721 Non-Fungible Token (NFT) contract. Its sole responsibility is to mint and manage the final Impact Tokens, which serve as immutable, verifiable certificates of social impact. It acts as a simple, secure digital ledger.1

This separation adheres to the Single Responsibility Principle. By isolating the complex, stateful logic of the escrow process from the standardized, relatively simple logic of the NFT, each contract becomes easier to understand, test, and secure. A monolithic contract combining both functionalities would be unnecessarily complex and more prone to error.1

### **2.2 Establishing Cryptographic Guarantees: The Gated Minting Pattern**

The connection between these two contracts establishes a powerful, programmatically enforced guarantee: an Impact Token can *only* be created as the final, automated step of a fully funded and verified project within the $ProjectEscrow contract. This is achieved through a "gated minting" pattern.1

In this pattern, the ability to mint new tokens in the $ImpactToken contract is not granted to a user or an admin wallet. Instead, it is exclusively assigned to the $ProjectEscrow contract itself. This creates an unbreakable, on-chain link between the verified completion of real-world work and the issuance of its digital representation. No one can circumvent the process to mint an unearned token, thereby ensuring the integrity and value of the Impact Tokens as verifiable assets.

### **2.3 A Technical Deep Dive into Ownable and Programmatic Ownership Transfer**

The technical implementation of the gated minting pattern relies on OpenZeppelin's Ownable.sol contract, an industry-standard component for access control.1 The

Ownable contract provides a simple yet powerful ownership model with three key features 15:

* A state variable, address public owner, which stores the address of the owner.  
* A modifier, onlyOwner, which can be applied to functions to restrict their execution to the owner address.  
* A function, transferOwnership(address newOwner), which allows the current owner to transfer ownership to a new address.

The critical feature that enables this architecture is that the newOwner passed to transferOwnership can be the address of another smart contract, not just a standard user wallet (Externally Owned Account).17

The implementation sequence will be as follows:

1. The $ImpactToken contract will be written to inherit from Ownable.sol.  
2. Its minting function will be protected by the onlyOwner modifier, ensuring only the contract's owner can create new tokens.14  
3. During deployment, a script will first deploy the $ImpactToken contract.  
4. It will then deploy the $ProjectEscrow contract.  
5. Finally, the script will execute a transaction that calls the transferOwnership function on the deployed $ImpactToken instance, passing the address of the $ProjectEscrow contract as the newOwner.

This final step programmatically establishes the gated minting mechanism, making the $ProjectEscrow contract the sole minter of Impact Tokens. This architectural choice is not just a secure pattern for the hackathon prototype; it is a fundamentally extensible and upgradable design. In the future, as the platform evolves to include features like a DAO for governance, the system can be upgraded without compromising the integrity of existing assets.1 A new, more advanced governance contract could be deployed, and the original

$ProjectEscrow contract could then transfer its ownership of the $ImpactToken contract to this new DAO. This allows the business logic to evolve while the $ImpactToken contract address remains constant, ensuring all previously minted tokens retain their history and validity—a hallmark of a production-ready mindset.

## **Section 3: Implementation of $ProjectEscrow.sol \- Part 1: Data Structures**

This section translates the conceptual model of a philanthropic project into concrete, gas-efficient Solidity data structures that will be stored on the blockchain.

### **3.1 Contract Scaffolding: Pragmas, Imports, and Security Guards**

The $ProjectEscrow.sol contract begins with standard declarations and imports from the OpenZeppelin library.

Solidity

// SPDX-License-Identifier: MIT  
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";  
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract ProjectEscrow is Ownable, ReentrancyGuard {  
    // Contract logic will be added here.  
}

* **pragma solidity ^0.8.24;**: Declares the compatible Solidity compiler version.  
* **import "@openzeppelin/contracts/access/Ownable.sol";**: Imports the Ownable contract to manage the privileged "Verifier" role. The contract deployer will initially be the owner.1  
* **import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";**: Imports a security utility that provides the nonReentrant modifier. This modifier will be applied to all functions that handle fund transfers to prevent re-entrancy attacks, a common and dangerous vulnerability in smart contracts.1

### **3.2 Defining State with Enums: The MilestoneState**

An enum (enumeration) is used to create a custom data type with a finite set of named values. This is clearer and less error-prone than using integers (e.g., 0, 1, 2\) to represent the state of a milestone.

Solidity

// Defines the possible states for a project milestone.  
enum MilestoneState {  
    Pending,  
    Verified,  
    Paid  
}

### **3.3 Core Data Schemas: The Milestone and Project Structs**

A struct is a custom data type that groups related variables, allowing for the creation of complex data records.21 The system uses two primary structs to model its data.

The Milestone struct holds the state for each individual project deliverable:

Solidity

// Represents a single, fundable milestone within a project.  
struct Milestone {  
    string description;  
    uint256 amount;  
    MilestoneState state;  
}

The Project struct encapsulates all data related to a single philanthropic initiative. It is the central data entity in the entire system.

Solidity

// Encapsulates all data for a single philanthropic project.  
struct Project {  
    uint256 projectId;  
    address payable creator; // The NGO's address.  
    address donor;  
    uint256 totalAmount;  
    uint256 fundsRaised;  
    Milestone milestones;  
    bool isComplete;  
}

The table below provides a detailed description of each field within the Project struct.

| Field Name | Data Type | Description |
| :---- | :---- | :---- |
| projectId | uint256 | A unique identifier for the project. |
| creator | address payable | The wallet address of the NGO, designated as payable to receive funds. |
| donor | address | The wallet address of the corporate donor who funds the project. |
| totalAmount | uint256 | The total funding required for the project, calculated as the sum of all milestone amounts. |
| fundsRaised | uint256 | The current amount of funds donated to and locked in the contract for this project. |
| milestones | Milestone | A dynamic array storing the details and state of each project milestone. |
| isComplete | bool | A flag that is set to true upon verification and payment of the final milestone. |

### **3.4 On-Chain State Management: The projects Mapping and projectCounter**

To store and manage the Project structs on-chain, the contract uses a mapping and a counter.

Solidity

// Mapping from a project ID to the Project struct.  
mapping(uint256 \=\> Project) public projects;

// Counter to generate unique project IDs.  
uint256 public projectCounter;

* **mapping(uint256 \=\> Project) public projects;**: A mapping is a key-value store. This mapping allows for efficient, O(1) lookup of any Project struct using its unique projectId. The public visibility keyword automatically creates a getter function, allowing external accounts to read project data.24  
* **uint256 public projectCounter;**: This state variable serves as a simple and gas-efficient mechanism to generate unique IDs for new projects and to track the total number of projects created on the platform.

## **Section 4: Implementation of $ProjectEscrow.sol \- Part 2: Core Functions**

This section covers the implementation of the initial state-changing functions that bring the platform to life: creating projects and accepting donations.

### **4.1 Function Analysis: createProject**

The createProject function allows for the creation of new philanthropic initiatives on the platform. For the hackathon demo, it is publicly callable.

Solidity

function createProject(  
    address \_ngo,  
    uint256 memory \_milestoneAmounts,  
    string memory \_milestoneDescriptions  
) external {  
    require(\_milestoneAmounts.length \== \_milestoneDescriptions.length, "Input array lengths must match");  
    require(\_milestoneAmounts.length \> 0, "Project must have at least one milestone");

    uint256 currentId \= projectCounter;  
    Project storage newProject \= projects\[currentId\];  
    newProject.projectId \= currentId;  
    newProject.creator \= payable(\_ngo);

    uint256 totalProjectAmount \= 0;  
    for (uint i \= 0; i \< \_milestoneAmounts.length; i++) {  
        newProject.milestones.push(  
            Milestone({  
                description: \_milestoneDescriptions\[i\],  
                amount: \_milestoneAmounts\[i\],  
                state: MilestoneState.Pending  
            })  
        );  
        totalProjectAmount \+= \_milestoneAmounts\[i\];  
    }

    newProject.totalAmount \= totalProjectAmount;  
    projectCounter++;  
}

The function signature, which accepts milestone data as two parallel arrays (uint256 and string) instead of a single array of Milestone structs, is a deliberate and professional design choice. The Solidity Application Binary Interface (ABI) has limitations regarding the passing of complex types like arrays of structs to external functions.25 The standard and most gas-efficient pattern is to deconstruct the data into parallel arrays of its constituent primitive types, as shown here. This demonstrates a practical understanding of the EVM's constraints.

The function logic proceeds as follows:

1. It validates that the input arrays are of equal and non-zero length.  
2. It retrieves the current value of projectCounter to use as the new project's unique ID.  
3. It creates a new Project struct in storage at that ID.  
4. It iterates through the input arrays, creating a Milestone struct for each entry, pushing it to the project's milestones array, and summing the amounts to calculate the totalAmount.  
5. Finally, it increments the projectCounter for the next project.

### **4.2 Function Analysis: donate**

The donate function enables a corporation (or any address) to fund a project. The payable modifier is essential, as it allows the function to receive MATIC (or the native currency of the blockchain) along with the transaction call.27 The amount of MATIC sent is accessible via the global

msg.value variable.

Solidity

function donate(uint256 \_projectId) external payable nonReentrant {  
    Project storage projectToFund \= projects\[\_projectId\];

    // Validation checks  
    require(projectToFund.creator\!= address(0), "Project does not exist");  
    require(projectToFund.fundsRaised \== 0, "Project is already funded");  
    require(msg.value \== projectToFund.totalAmount, "Donation must match the total project amount");

    // Update project state  
    projectToFund.fundsRaised \= msg.value;  
    projectToFund.donor \= msg.sender;  
}

This function implements several critical validation steps using require statements:

* It ensures the project exists by checking that its creator address is not the zero address.  
* For the simplicity of the hackathon demo, it enforces that a project can only be funded once by checking if fundsRaised is zero.  
* It verifies that the amount of MATIC sent (msg.value) exactly matches the project's required totalAmount.

If all checks pass, the function updates the project's state to record the amount raised and the address of the donor (msg.sender). The nonReentrant modifier from OpenZeppelin's ReentrancyGuard is applied to protect this function from re-entrancy attacks. The received MATIC is automatically and securely held by the contract itself.

## **Section 5: Foundational Validation: Unit Testing the Core Logic**

Automated unit tests are non-negotiable for smart contract development. They verify the correctness of the on-chain logic, ensuring a reliable foundation before deploying to a live network. Hardhat's testing environment uses the Mocha framework and Chai assertion library by default.6

### **5.1 Structuring the Test Suite with Mocha and Chai**

Tests will be written in a file named /test/ProjectEscrow.test.js. The structure uses describe() blocks to group related tests and it() blocks to define individual test cases. The Chai library provides the expect syntax for making assertions about the contract's state and behavior.6

### **5.2 The beforeEach Hook: Ensuring a Clean State for Each Test**

To ensure that tests are independent and do not influence one another, a beforeEach hook is used. This function runs before every single it() test case, deploying a fresh instance of the $ProjectEscrow contract. This practice is essential for preventing state leakage between tests and creating a reliable, non-flaky test suite.7

JavaScript

const { expect } \= require("chai");  
const { ethers } \= require("hardhat");

describe("ProjectEscrow", function () {  
  let ProjectEscrow, projectEscrow, owner, ngo, donor;

  beforeEach(async function () {  
    \[owner, ngo, donor\] \= await ethers.getSigners();  
    ProjectEscrow \= await ethers.getContractFactory("ProjectEscrow");  
    projectEscrow \= await ProjectEscrow.deploy(owner.address);  
  });

  // Test cases will be added here.  
});

### **5.3 Test Case 1: Validating Successful Project Creation**

This test verifies that the createProject function correctly initializes a new project with the specified parameters.

JavaScript

it("Should allow creation of a new project", async function () {  
  const milestoneAmounts \= \[ethers.parseEther("0.5"), ethers.parseEther("0.5")\];  
  const milestoneDescriptions \= \["Milestone 1", "Milestone 2"\];  
  const totalAmount \= ethers.parseEther("1.0");

  await projectEscrow.connect(owner).createProject(  
    ngo.address,  
    milestoneAmounts,  
    milestoneDescriptions  
  );

  const project \= await projectEscrow.projects(0);

  expect(project.projectId).to.equal(0);  
  expect(project.creator).to.equal(ngo.address);  
  expect(project.totalAmount).to.equal(totalAmount);  
  expect(project.milestones.length).to.equal(2);  
  expect(await projectEscrow.projectCounter()).to.equal(1);  
});

### **5.4 Test Case 2: Verifying Donation and Fund Locking Mechanisms**

This test confirms that the donate function correctly records the donation and, critically, that the contract itself securely locks the funds.

JavaScript

it("Should allow a donor to fund a project", async function () {  
  const milestoneAmounts \= \[ethers.parseEther("1.0")\];  
  const milestoneDescriptions \= \["Complete Project"\];  
  const totalAmount \= ethers.parseEther("1.0");

  await projectEscrow.connect(owner).createProject(  
    ngo.address,  
    milestoneAmounts,  
    milestoneDescriptions  
  );

  const contractAddress \= await projectEscrow.getAddress();  
    
  await expect(() \=\>   
    donor.sendTransaction({  
      to: contractAddress,  
      value: totalAmount,  
      data: projectEscrow.interface.encodeFunctionData("donate", )  
    })  
  ).to.changeEtherBalance(projectEscrow, totalAmount);

  const project \= await projectEscrow.projects(0);  
  expect(project.fundsRaised).to.equal(totalAmount);  
  expect(project.donor).to.equal(donor.address);  
});

This test uses changeEtherBalance from the hardhat-chai-matchers library (included in hardhat-toolbox) to assert that the contract's balance increased by the exact donation amount. This provides direct, cryptographic proof that the funds are locked in the escrow as intended.

A robust test suite must validate not only successful execution but also expected failures. The most important security features of a smart contract are often the require statements that prevent invalid state transitions. Testing these reverts is a critical, security-oriented practice. Hardhat and Chai provide a specific pattern for this using revertedWith.6 The following tests should also be implemented to ensure the

donate function is properly secured:

JavaScript

it("Should revert if donating to a non-existent project", async function () {  
  await expect(  
    projectEscrow.connect(donor).donate(99, { value: ethers.parseEther("1.0") })  
  ).to.be.revertedWith("Project does not exist");  
});

it("Should revert if donation amount is incorrect", async function () {  
  const milestoneAmounts \= \[ethers.parseEther("1.0")\];  
  const milestoneDescriptions \= \["Complete Project"\];

  await projectEscrow.connect(owner).createProject(  
    ngo.address,  
    milestoneAmounts,  
    milestoneDescriptions  
  );

  await expect(  
    projectEscrow.connect(donor).donate(0, { value: ethers.parseEther("0.5") })  
  ).to.be.revertedWith("Donation must match the total project amount");  
});

By including these negative test cases, the development team demonstrates a professional, security-first approach that goes beyond simply testing the "happy path."

## **Conclusion: A Robust Foundation for Day 2 and Beyond**

The successful completion of the Day 1 tasks yields a fully configured development environment, a secure and scalable two-contract architecture, and a tested on-chain mechanism for project creation and funding. This solid foundation significantly de-risks the subsequent days of the hackathon. With the core on-chain logic validated, the team can proceed with confidence to implement the verification loop, finalize the smart contract suite, and begin building the user-facing frontend application as outlined for Day 2\.1

#### **Works cited**

1. Hackathon Demo Roadmap\_ ImpactChain.pdf  
2. Hardhat \- Polygon Knowledge Layer, accessed September 12, 2025, [https://docs.polygon.technology/tools/dApp-development/common-tools/hardhat/](https://docs.polygon.technology/tools/dApp-development/common-tools/hardhat/)  
3. How to Code and Deploy a Polygon Smart Contract | Alchemy Docs, accessed September 12, 2025, [https://www.alchemy.com/docs/how-to-code-and-deploy-a-polygon-smart-contract](https://www.alchemy.com/docs/how-to-code-and-deploy-a-polygon-smart-contract)  
4. Deploying a smart contract on the Polygon test network \- Hashnode, accessed September 12, 2025, [https://adityamca123.hashnode.dev/deploying-a-smart-contract-on-the-polygon-test-network](https://adityamca123.hashnode.dev/deploying-a-smart-contract-on-the-polygon-test-network)  
5. How to Get Started with Developing on Polygon?, accessed September 12, 2025, [https://forum.polygon.technology/t/how-to-get-started-with-developing-on-polygon/19803](https://forum.polygon.technology/t/how-to-get-started-with-developing-on-polygon/19803)  
6. How to Unit Test a Smart Contract | Alchemy Docs, accessed September 12, 2025, [https://www.alchemy.com/docs/how-to-unit-test-a-smart-contract](https://www.alchemy.com/docs/how-to-unit-test-a-smart-contract)  
7. Hardhat Guide: Automated Smart Contract Testing, accessed September 12, 2025, [https://www.krayondigital.com/blog/hardhat-guide-automated-smart-contract-testing](https://www.krayondigital.com/blog/hardhat-guide-automated-smart-contract-testing)  
8. Deploy a Smart Contract on Polygon Mainnet with AMB Access and Hardhat Ignition, accessed September 12, 2025, [https://repost.aws/articles/ARMiTkQJ-GRaqeDCxHVIoPhA/deploy-a-smart-contract-on-polygon-mainnet-with-amb-access-and-hardhat-ignition](https://repost.aws/articles/ARMiTkQJ-GRaqeDCxHVIoPhA/deploy-a-smart-contract-on-polygon-mainnet-with-amb-access-and-hardhat-ignition)  
9. Verify Smart Contract on Polygonscan— using Hardhat \- CoinsBench, accessed September 12, 2025, [https://coinsbench.com/verify-smart-contract-on-polygonscan-using-hardhat-9b8331dbd888](https://coinsbench.com/verify-smart-contract-on-polygonscan-using-hardhat-9b8331dbd888)  
10. Polygon Mumbai Faucet \- Free Testnet POL Tokens, accessed September 12, 2025, [https://faucet.quicknode.com/polygon/mumbai](https://faucet.quicknode.com/polygon/mumbai)  
11. How to Get Free Testnet MATIC Tokens using Mumbai Testnet Faucet on Polygon?, accessed September 12, 2025, [https://www.remote3.co/blog-post/how-to-get-free-testnet-matic-tokens-using-mumbai-testnet-faucet-on-polygon](https://www.remote3.co/blog-post/how-to-get-free-testnet-matic-tokens-using-mumbai-testnet-faucet-on-polygon)  
12. Polygon Faucet, accessed September 12, 2025, [https://faucet.polygon.technology/](https://faucet.polygon.technology/)  
13. Faucets \- Get Testnet ETH and Matic Tokens \- Alchemy, accessed September 12, 2025, [https://www.alchemy.com/faucets](https://www.alchemy.com/faucets)  
14. How to only allow contract owner to mint token? \- Ethereum Stack Exchange, accessed September 12, 2025, [https://ethereum.stackexchange.com/questions/124119/how-to-only-allow-contract-owner-to-mint-token](https://ethereum.stackexchange.com/questions/124119/how-to-only-allow-contract-owner-to-mint-token)  
15. Understanding Ownership Transfer in Solidity Smart Contracts | by Holiday \- Medium, accessed September 12, 2025, [https://i-abdulsalihu.medium.com/understanding-ownership-transfer-in-solidity-smart-contractshow-to-transfer-ownership-in-your-smart-63cea8d1572c](https://i-abdulsalihu.medium.com/understanding-ownership-transfer-in-solidity-smart-contractshow-to-transfer-ownership-in-your-smart-63cea8d1572c)  
16. Ownership \- OpenZeppelin Docs, accessed September 12, 2025, [https://docs.openzeppelin.com/contracts/2.x/api/ownership](https://docs.openzeppelin.com/contracts/2.x/api/ownership)  
17. Access Control \- OpenZeppelin Docs, accessed September 12, 2025, [https://docs.openzeppelin.com/contracts/4.x/access-control](https://docs.openzeppelin.com/contracts/4.x/access-control)  
18. How to call the function of another smart contract in Solidity? With an example of OpenZeppelin transferOwnership function, accessed September 12, 2025, [https://forum.openzeppelin.com/t/how-to-call-the-function-of-another-smart-contract-in-solidity-with-an-example-of-openzeppelin-transferownership-function/34409](https://forum.openzeppelin.com/t/how-to-call-the-function-of-another-smart-contract-in-solidity-with-an-example-of-openzeppelin-transferownership-function/34409)  
19. Access Control \- OpenZeppelin Docs, accessed September 12, 2025, [https://docs.openzeppelin.com/contracts/5.x/access-control](https://docs.openzeppelin.com/contracts/5.x/access-control)  
20. Access Control \- OpenZeppelin Docs, accessed September 12, 2025, [https://docs.openzeppelin.com/contracts/2.x/access-control](https://docs.openzeppelin.com/contracts/2.x/access-control)  
21. How do Solidity structs work? | Alchemy Docs, accessed September 12, 2025, [https://www.alchemy.com/docs/how-do-solidity-structs-work](https://www.alchemy.com/docs/how-do-solidity-structs-work)  
22. Day 10: Solidity Arrays, Structs, and Mappings | by Gurnoor Singh | CoinsBench, accessed September 12, 2025, [https://coinsbench.com/day-10-solidity-arrays-structs-and-mappings-377a8e08842d](https://coinsbench.com/day-10-solidity-arrays-structs-and-mappings-377a8e08842d)  
23. Structs in Solidity. 100 days of solidity (Day 13–17) | by Favorite\_blockchain\_lady | Coinmonks | Medium, accessed September 12, 2025, [https://medium.com/coinmonks/structs-in-solidity-924e1f2e38cc](https://medium.com/coinmonks/structs-in-solidity-924e1f2e38cc)  
24. Introduction to Smart Contracts — Solidity 0.8.31 documentation, accessed September 12, 2025, [https://docs.soliditylang.org/en/latest/introduction-to-smart-contracts.html](https://docs.soliditylang.org/en/latest/introduction-to-smart-contracts.html)  
25. Can solidity have a function signature that accepts an object array as a parameter? \- Reddit, accessed September 12, 2025, [https://www.reddit.com/r/ethdev/comments/u8a87n/can\_solidity\_have\_a\_function\_signature\_that/](https://www.reddit.com/r/ethdev/comments/u8a87n/can_solidity_have_a_function_signature_that/)  
26. pass an array as parameter in solidity \- Ethereum Stack Exchange, accessed September 12, 2025, [https://ethereum.stackexchange.com/questions/58464/pass-an-array-as-parameter-in-solidity](https://ethereum.stackexchange.com/questions/58464/pass-an-array-as-parameter-in-solidity)  
27. Payable Functions | By RareSkills, accessed September 12, 2025, [https://rareskills.io/learn-solidity/payable-functions](https://rareskills.io/learn-solidity/payable-functions)  
28. The Payable Modifier \- Ethereum Blockchain Developer, accessed September 12, 2025, [https://www.ethereum-blockchain-developer.com/courses/ethereum-course-2024/project-smart-money---deposit-and-withdrawals/the-payable-modifier-and-msg.value](https://www.ethereum-blockchain-developer.com/courses/ethereum-course-2024/project-smart-money---deposit-and-withdrawals/the-payable-modifier-and-msg.value)  
29. What are Payable Functions in Solidity? | Alchemy Docs, accessed September 12, 2025, [https://www.alchemy.com/docs/solidity-payable-functions](https://www.alchemy.com/docs/solidity-payable-functions)  
30. Exploring the Differences Between Payable and Non-Payable Functions in Solidity: An In-Depth Analysis | by Oluwatosin Serah | Coinmonks | Medium, accessed September 12, 2025, [https://medium.com/coinmonks/exploring-the-differences-between-payable-and-non-payable-functions-in-solidity-an-in-depth-d031c6ae577b](https://medium.com/coinmonks/exploring-the-differences-between-payable-and-non-payable-functions-in-solidity-an-in-depth-d031c6ae577b)  
31. Unit Testing with Hardhat \- Earn \- StackUp, accessed September 12, 2025, [https://earn.stackup.dev/learn/pathways/web3-development/skills/web3-development-frameworks/modules/hardhat-for-web3-development/tutorials/unit-testing-with-hardhat](https://earn.stackup.dev/learn/pathways/web3-development/skills/web3-development-frameworks/modules/hardhat-for-web3-development/tutorials/unit-testing-with-hardhat)  
32. The Correct Way To Write Tests For Your Smart Contracts, Using HardHat And Ethers-JS: Reentrancy Attacks | by Bashorun Dolapo | CoinsBench, accessed September 12, 2025, [https://coinsbench.com/the-correct-way-to-write-tests-for-your-smart-contracts-using-hardhat-and-ethers-js-reentrancy-5438006e90a0](https://coinsbench.com/the-correct-way-to-write-tests-for-your-smart-contracts-using-hardhat-and-ethers-js-reentrancy-5438006e90a0)