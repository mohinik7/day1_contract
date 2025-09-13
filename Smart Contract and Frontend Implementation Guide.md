

# **ImpactChain Hackathon: A Developer's Guide to Day 2 Implementation**

## **Introduction: Bridging On-Chain Logic with the User Interface**

Day 1 of the hackathon established the foundational architecture of the ImpactChain smart contract system, defining the core data structures and initial functions within a robust Hardhat development environment.1 Day 2 marks a pivotal transition. The objective is to complete the system's on-chain logic and, for the first time, bridge this backend intelligence to an interactive frontend. This phase moves the project from a collection of Solidity files into the skeleton of a living, breathing decentralized application (dApp).

The day's work is structured around three primary, interconnected objectives, as outlined in the project's technical roadmap.1 First, the smart contract suite will be finalized. This involves implementing the

$ImpactToken$ non-fungible token (NFT) contract, which represents the verifiable impact asset, and integrating the critical verification and fund-release mechanisms into the $ProjectEscrow$ contract. Second, the foundational "scaffolding" of the frontend application will be constructed. The focus here is not on aesthetics but on core functionality: establishing a secure and reliable connection to the blockchain via a user's wallet, specifically MetaMask. Finally, these two streams of work will be unified through an automated deployment script. This script will handle the complex, multi-step process of deploying the interconnected contracts to the Polygon Mumbai testnet and configuring their relationship to ensure the system's integrity.

This guide provides a detailed, step-by-step implementation plan for these tasks. It will utilize the established technical stack of Solidity for smart contracts, OpenZeppelin for secure, standardized components, Hardhat for the development and deployment framework, and Ethers.js for frontend-to-blockchain communication within a React or Vue application.1

## **Part 1: Implementing the ImpactToken NFT Contract**

The $ImpactToken$ contract is more than just a digital collectible; it is the tangible, cryptographic proof of a completed and verified social impact initiative. Its implementation must be secure, standardized, and, most importantly, its creation must be strictly controlled to maintain its value as a verifiable asset. This section provides the complete code and a deep-dive analysis for this critical ERC-721 contract.

### **The Anatomy of a Gated ERC-721 Contract**

The implementation strategy for the $ImpactToken$ contract prioritizes security and interoperability by leveraging industry-standard components from the OpenZeppelin library.1 This is a deliberate architectural choice. Rather than writing an ERC-721 implementation from scratch, which is time-consuming and prone to subtle but critical vulnerabilities, the project will import OpenZeppelin's battle-tested and professionally audited

ERC721.sol and Ownable.sol contracts.2 This approach not only accelerates development but also ensures that the resulting token is fully compliant with the ERC-721 standard, making it compatible with all major NFT wallets, block explorers, and future marketplaces.

The access control for this contract is managed using the Ownable.sol pattern. This contract provides a simple yet powerful access control mechanism, establishing a single privileged account designated as the owner.5 By default, the account that deploys the contract becomes its initial owner. The

Ownable contract provides an onlyOwner modifier, which can be applied to functions to restrict their execution to this owner address exclusively. This modifier is the technical foundation of the "gated minting" system, a core requirement of the ImpactChain platform.1

### **Full Contract Code and Analysis: ImpactToken.sol**

The following is the complete, commented Solidity code for the $ImpactToken.sol$ contract.

Solidity

// SPDX-License-Identifier: MIT  
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";  
import "@openzeppelin/contracts/access/Ownable.sol";

/\*\*  
 \* @title ImpactToken  
 \* @dev An ERC721 non-fungible token contract that represents a verifiable  
 \* impact certificate. Minting is restricted to the contract owner, which  
 \* will be the ProjectEscrow contract.  
 \*/  
contract ImpactToken is ERC721, Ownable {  
    /\*\*  
     \* @dev Initializes the contract, setting the token name and symbol,  
     \* and establishing the deployer as the initial owner.  
     \*/  
    constructor()  
        ERC721("ImpactChain Certificate", "ICC")  
        Ownable(msg.sender) // Explicitly set initial owner in OZ 5.x  
    {}

    /\*\*  
     \* @dev Mints a new Impact Token and assigns it to a recipient.  
     \* This function is protected by the \`onlyOwner\` modifier, ensuring that  
     \* only the designated owner (the ProjectEscrow contract) can create new tokens.  
     \* It also sets the token's metadata URI.  
     \* @param to The address to receive the newly minted token.  
     \* @param tokenId The unique identifier for the new token.  
     \* @param tokenURI The URI pointing to the token's metadata JSON file.  
     \*/  
    function safeMint(address to, uint256 tokenId, string memory tokenURI)  
        external  
        onlyOwner  
    {  
        \_safeMint(to, tokenId);  
        \_setTokenURI(tokenId, tokenURI);  
    }  
}

#### **Line-by-Line Breakdown**

* pragma solidity ^0.8.20;: This line specifies the version of the Solidity compiler to be used, ensuring that the code is compiled with a modern and stable version.  
* import "@openzeppelin/contracts/token/ERC721/ERC721.sol";: This imports the standard implementation of the ERC-721 NFT standard from the OpenZeppelin library, providing all the core functionality for tracking token ownership and transfers.3  
* import "@openzeppelin/contracts/access/Ownable.sol";: This imports the access control module that provides the ownership pattern and the onlyOwner modifier.6  
* contract ImpactToken is ERC721, Ownable {... }: This line declares the contract and specifies that it inherits from both the ERC721 and Ownable contracts. This means $ImpactToken$ gains all the functions and modifiers defined in those parent contracts.  
* constructor() ERC721("ImpactChain Certificate", "ICC") Ownable(msg.sender) {}: The constructor is a special function that runs only once when the contract is deployed. It initializes the ERC721 component with the token's full name and its ticker symbol, as specified in the roadmap.1 It also calls the  
  Ownable constructor, explicitly setting the initial owner to msg.sender—the address of the account deploying the contract.7  
* function safeMint(address to, uint256 tokenId, string memory tokenURI) external onlyOwner {... }: This is the most critical function in the contract and serves as the sole entry point for creating new tokens.  
  * external: This visibility specifier means the function can be called from other contracts and via external transactions.  
  * onlyOwner: This modifier, inherited from Ownable.sol, is the gatekeeper. Before any code inside this function is executed, the contract checks if the caller (msg.sender) is the current owner. If not, the transaction reverts. This is the programmatic enforcement of the "gated minting" rule.6  
  * \_safeMint(to, tokenId);: This is an internal function from the parent ERC721.sol contract that performs the actual minting operation. It creates a new token with the specified tokenId and assigns its ownership to the to address.2 The choice of  
    \_safeMint over the simpler \_mint is a deliberate security best practice. \_safeMint includes an additional check to ensure that if the recipient (to) is another smart contract, that contract implements the IERC721Receiver interface, which signals its ability to handle NFT transfers. This prevents tokens from being accidentally sent to contracts that cannot manage them, which would result in the tokens being permanently locked and lost.8 While the hackathon's "happy path" involves minting to a corporate user's standard wallet, adhering to this safer standard demonstrates a "security first" mindset, a core design principle of the project.1  
  * \_setTokenURI(tokenId, tokenURI);: After minting, this function from the ERC721 metadata extension is called to associate the new token with its metadata. The tokenURI is a URL that points to a JSON file containing the NFT's name, description, image, and other attributes.9

The "gated minting" pattern implemented here is the technical expression of the project's core value proposition: creating "cryptographic truth".1 An Impact Token has value only if its existence proves that a verified impact occurred. The

onlyOwner modifier on the safeMint function is the mechanism for this enforcement. When the ownership of this $ImpactToken$ contract is later transferred to the address of the deployed $ProjectEscrow$ contract, an unbreakable, on-chain link is forged. From that moment on, the code itself becomes the guarantee. No administrator, developer, or malicious actor can create a "fake" Impact Token because the safeMint function will only ever respond to calls originating from the $ProjectEscrow$ contract. This architecture transforms a business rule—"tokens are only created upon final project verification"—into an immutable law of the system, a powerful and compelling point to demonstrate.

## **Part 2: Finalizing the ProjectEscrow Contract Logic**

With the $ImpactToken$ contract defined, the next step is to complete the logic within the $ProjectEscrow$ contract. This involves implementing the verification-to-payout loop, which serves as the engine of the ImpactChain platform. This logic will enable the designated "Verifier" to approve project milestones, which in turn will automatically trigger the release of funds to the NGO and, upon final completion, the minting of the Impact Token for the donor.

### **Architecting the Contract-to-Contract Link**

For the $ProjectEscrow$ contract to mint an NFT, it must first be aware of the $ImpactToken$ contract's location on the blockchain and have a way to call its safeMint function. This link is established at the moment of deployment.

First, a minimal interface for the $ImpactToken$ contract is defined. This allows the Solidity compiler to understand the shape of the external safeMint function and perform type-checking.

Solidity

interface IImpactToken {  
    function safeMint(address to, uint256 tokenId, string memory tokenURI) external;  
}

Next, a new state variable is added to $ProjectEscrow.sol$ to store the address of the $ImpactToken$ contract. This variable is declared as immutable, which is a critical optimization and security feature. An immutable variable can only be assigned a value once, within the contract's constructor, and cannot be changed thereafter. This guarantees that the link between the escrow and token contracts is permanent and tamper-proof, while also saving gas costs compared to a regular storage variable.

Solidity

// In ProjectEscrow.sol  
IImpactToken public immutable impactTokenContract;

Finally, the constructor of $ProjectEscrow.sol$ is modified to accept the deployed address of the $ImpactToken$ contract as an argument. This address is then stored in the impactTokenContract state variable, permanently linking the two contracts.1

Solidity

// In ProjectEscrow.sol constructor  
constructor(address \_impactTokenAddress) {  
    impactTokenContract \= IImpactToken(\_impactTokenAddress);  
    //... other constructor logic  
}

### **The Verification Mechanism: Simulating the Oracle**

The verifyMilestone function acts as the stand-in for the complex AI oracle proposed in the full project vision. For the hackathon, this function is restricted to the contract's owner, who plays the role of the trusted verifier.1

Solidity

// In ProjectEscrow.sol  
function verifyMilestone(uint256 \_projectId, uint256 \_milestoneIndex)  
    external  
    onlyOwner  
    nonReentrant  
{  
    Project storage project \= projects\[\_projectId\];  
      
    // Input and state validation  
    require(project.creator\!= address(0), "Project does not exist");  
    require(\_milestoneIndex \< project.milestones.length, "Invalid milestone index");  
      
    Milestone storage milestone \= project.milestones\[\_milestoneIndex\];  
    require(milestone.state \== MilestoneState.Pending, "Milestone not pending verification");

    // Update state before external call  
    milestone.state \= MilestoneState.Verified;

    // Trigger the fund release  
    \_releaseFunds(\_projectId, \_milestoneIndex);  
}

This function's logic is straightforward but secure. The onlyOwner modifier ensures only the designated verifier can call it. It then performs a series of require checks to validate the inputs and ensure the milestone is in the correct state (Pending) before proceeding. It updates the milestone's state to Verified and then calls the internal \_releaseFunds function to handle the payout.

### **The Automated Payout: Fund Release and Minting Trigger**

The \_releaseFunds function contains the core automation logic of the platform. It handles the secure transfer of funds and, upon project completion, triggers the minting of the Impact Token.

Solidity

// In ProjectEscrow.sol  
function \_releaseFunds(uint256 \_projectId, uint256 \_milestoneIndex) internal {  
    Project storage project \= projects\[\_projectId\];  
    Milestone storage milestone \= project.milestones\[\_milestoneIndex\];

    // Securely transfer funds to the NGO (creator)  
    (bool success, ) \= project.creator.call{value: milestone.amount}("");  
    require(success, "Fund transfer failed.");

    // Update milestone state to Paid  
    milestone.state \= MilestoneState.Paid;

    // Check if this is the final milestone  
    if (\_milestoneIndex \== project.milestones.length \- 1\) {  
        project.isComplete \= true;

        // Construct the token URI  
        string memory tokenURI \= \_constructTokenURI(\_projectId);

        // Mint the Impact Token NFT to the donor  
        impactTokenContract.safeMint(project.donor, \_projectId, tokenURI);  
    }  
}

function \_constructTokenURI(uint256 \_projectId) internal pure returns (string memory) {  
    // For the demo, this will point to a generic JSON file.  
    // In production, this would be a unique, dynamically generated URI.  
    // Example: return string(abi.encodePacked("https://api.impactchain.com/metadata/", Strings.toString(\_projectId)));  
    return "https://api.jsonbin.io/v3/b/6... (replace with your hosted JSON)";  
}

The visibility of this function is set to internal, which is a critical security boundary. An internal function can only be called by other functions within the same contract or by contracts that inherit from it. It cannot be called externally. This design creates a secure "funnel," ensuring that the only way to trigger a fund release is through a valid, authorized call to a public function like verifyMilestone. If \_releaseFunds were public, an attacker could attempt to call it directly, bypassing the entire verification process.

For the fund transfer itself, the code uses the low-level call method: (bool success, ) \= project.creator.call{value: milestone.amount}("");. This is the currently recommended best practice for sending Ether, as it is more robust against future changes in transaction gas costs than the older .transfer() or .send() methods. It forwards all available gas and returns a boolean indicating success, which is then checked with a require statement.

The climax of the on-chain logic occurs when the final milestone is verified. The contract initiates an external call to the impactTokenContract, invoking its safeMint function. It passes the original project.donor address as the recipient and the unique \_projectId as the tokenId. This single, atomic transaction—which encompasses state changes (Verified, Paid, isComplete), a value transfer (to the NGO), and an external call to mint an NFT—is the source of the system's "trustless" nature. Participants do not need to trust an administrator to process payments after verification; the smart contract guarantees that the act of verification *is* the trigger for payment and reward. This programmatic certainty eliminates human bias, delays, and the need for intermediaries.

### **Complete Refined Code: ProjectEscrow.sol**

The following is the fully refined source code for $ProjectEscrow.sol$, incorporating the logic from both Day 1 and Day 2\.

Solidity

// SPDX-License-Identifier: MIT  
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/Ownable.sol";  
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

// Minimal interface for the ImpactToken contract  
interface IImpactToken {  
    function safeMint(address to, uint256 tokenId, string memory tokenURI) external;  
}

contract ProjectEscrow is Ownable, ReentrancyGuard {  
    // The linked ImpactToken contract, set at deployment  
    IImpactToken public immutable impactTokenContract;

    // Enum for milestone states  
    enum MilestoneState { Pending, Verified, Paid }

    // Struct to define a single project milestone  
    struct Milestone {  
        string description;  
        uint256 amount;  
        MilestoneState state;  
    }

    // Struct to encapsulate all project data  
    struct Project {  
        uint256 projectId;  
        address payable creator; // NGO address  
        address donor;  
        uint256 totalAmount;  
        uint256 fundsRaised;  
        Milestone milestones;  
        bool isComplete;  
    }

    // State variables  
    mapping(uint256 \=\> Project) public projects;  
    uint256 public projectCounter;

    // Events  
    event ProjectCreated(uint256 indexed projectId, address indexed creator, uint256 totalAmount);  
    event DonationReceived(uint256 indexed projectId, address indexed donor, uint256 amount);  
    event MilestoneVerified(uint256 indexed projectId, uint256 indexed milestoneIndex);  
    event FundsReleased(uint256 indexed projectId, uint256 indexed milestoneIndex, uint256 amount);

    /\*\*  
     \* @dev Initializes the contract, setting the owner and the address  
     \* of the associated ImpactToken contract.  
     \* @param \_impactTokenAddress The deployed address of the ImpactToken contract.  
     \*/  
    constructor(address \_impactTokenAddress) Ownable(msg.sender) {  
        require(\_impactTokenAddress\!= address(0), "Invalid token contract address");  
        impactTokenContract \= IImpactToken(\_impactTokenAddress);  
    }

    /\*\*  
     \* @dev Creates a new philanthropic project.  
     \*/  
    function createProject(  
        address payable \_ngo,  
        uint256 memory \_milestoneAmounts,  
        string memory \_milestoneDescriptions  
    ) external {  
        require(\_ngo\!= address(0), "Invalid NGO address");  
        require(\_milestoneAmounts.length \== \_milestoneDescriptions.length, "Input array length mismatch");  
        require(\_milestoneAmounts.length \> 0, "Project must have at least one milestone");

        uint256 projectId \= projectCounter;  
        Project storage newProject \= projects\[projectId\];  
        newProject.projectId \= projectId;  
        newProject.creator \= \_ngo;

        uint256 totalProjectAmount \= 0;  
        for (uint i \= 0; i \< \_milestoneAmounts.length; i++) {  
            require(\_milestoneAmounts\[i\] \> 0, "Milestone amount must be positive");  
            newProject.milestones.push(Milestone({  
                description: \_milestoneDescriptions\[i\],  
                amount: \_milestoneAmounts\[i\],  
                state: MilestoneState.Pending  
            }));  
            totalProjectAmount \+= \_milestoneAmounts\[i\];  
        }  
        newProject.totalAmount \= totalProjectAmount;  
          
        projectCounter++;  
        emit ProjectCreated(projectId, \_ngo, totalProjectAmount);  
    }

    /\*\*  
     \* @dev Allows a donor to fund a project.  
     \*/  
    function donate(uint256 \_projectId) external payable nonReentrant {  
        Project storage project \= projects\[\_projectId\];  
        require(project.creator\!= address(0), "Project does not exist");  
        require(project.donor \== address(0), "Project already funded");  
        require(msg.value \== project.totalAmount, "Donation must match total project amount");

        project.fundsRaised \= msg.value;  
        project.donor \= msg.sender;

        emit DonationReceived(\_projectId, msg.sender, msg.value);  
    }

    /\*\*  
     \* @dev Verifies a milestone, acting as the mocked AI oracle. Only callable by the owner.  
     \*/  
    function verifyMilestone(uint256 \_projectId, uint256 \_milestoneIndex)  
        external  
        onlyOwner  
        nonReentrant  
    {  
        Project storage project \= projects\[\_projectId\];  
        require(project.creator\!= address(0), "Project does not exist");  
        require(\_milestoneIndex \< project.milestones.length, "Invalid milestone index");  
          
        Milestone storage milestone \= project.milestones\[\_milestoneIndex\];  
        require(milestone.state \== MilestoneState.Pending, "Milestone not pending verification");

        milestone.state \= MilestoneState.Verified;  
        emit MilestoneVerified(\_projectId, \_milestoneIndex);

        \_releaseFunds(\_projectId, \_milestoneIndex);  
    }

    /\*\*  
     \* @dev Internal function to release funds and trigger minting.  
     \*/  
    function \_releaseFunds(uint256 \_projectId, uint256 \_milestoneIndex) internal {  
        Project storage project \= projects\[\_projectId\];  
        Milestone storage milestone \= project.milestones\[\_milestoneIndex\];

        (bool success, ) \= project.creator.call{value: milestone.amount}("");  
        require(success, "Fund transfer failed.");

        milestone.state \= MilestoneState.Paid;  
        emit FundsReleased(\_projectId, \_milestoneIndex, milestone.amount);

        if (\_milestoneIndex \== project.milestones.length \- 1\) {  
            project.isComplete \= true;  
            string memory tokenURI \= \_constructTokenURI(\_projectId);  
            impactTokenContract.safeMint(project.donor, \_projectId, tokenURI);  
        }  
    }

    /\*\*  
     \* @dev Internal helper to construct the token URI.  
     \*/  
    function \_constructTokenURI(uint256 \_projectId) internal pure returns (string memory) {  
        // This should be replaced with a real metadata service URL.  
        return "https://api.jsonbin.io/v3/b/6... (replace with your hosted JSON)";  
    }  
}

## **Part 3: Scaffolding the Frontend and Wallet Integration**

With the on-chain logic complete and ready for deployment, the focus shifts to the user-facing application. The first and most critical step is to establish the connection between the browser-based frontend and the blockchain. This involves integrating the Ethers.js library and building the logic for a user to connect their MetaMask wallet.

### **Setting Up the Ethers.js Environment**

Ethers.js is a complete and compact JavaScript library for interacting with the Ethereum blockchain and its ecosystem.11 It provides the tools necessary to read data from the blockchain, query smart contract states, and, most importantly, request that a user sign and send transactions. To add it to the frontend project (e.g., a React application), run the following command in the project's root directory:

Bash

npm install ethers

### **Architecting a Blockchain Service Module**

To maintain a clean and scalable codebase, all direct blockchain interactions should be encapsulated within a dedicated service module (e.g., frontend/src/services/blockchainService.js). This approach adheres to the principle of separation of concerns: UI components remain responsible for rendering views and handling user input, while the service module manages the complexities of blockchain communication. This makes the code easier to debug, maintain, and test.

### **Implementing the "Connect Wallet" Component and Logic**

The "Connect Wallet" button is the user's entry point into the dApp. Its implementation involves detecting the user's wallet provider (MetaMask), requesting permission to access their accounts, and then setting up the necessary Ethers.js objects to enable blockchain interactions.

The following is a sample implementation for a React component, ConnectWalletButton.js, which also demonstrates how to use React's Context API for simple global state management—ideal for a hackathon.

**AppContext.js (Global State Management)**

JavaScript

import React, { useState, createContext } from 'react';

export const AppContext \= createContext();

export const AppProvider \= ({ children }) \=\> {  
    const \[provider, setProvider\] \= useState(null);  
    const \= useState(null);  
    const \[account, setAccount\] \= useState(null);

    return (  
        \<AppContext.Provider value\={{  
            provider, setProvider,  
            signer, setSigner,  
            account, setAccount  
        }}\>  
            {children}  
        \</AppContext.Provider\>  
    );  
};

**ConnectWalletButton.js (The Component)**

JavaScript

import React, { useContext, useEffect } from 'react';  
import { ethers } from 'ethers';  
import { AppContext } from './AppContext';

const ConnectWalletButton \= () \=\> {  
    const { setProvider, setSigner, setAccount, account } \= useContext(AppContext);

    const connectWallet \= async () \=\> {  
        if (typeof window.ethereum \=== 'undefined') {  
            alert("Please install MetaMask to use this dApp.");  
            return;  
        }

        try {  
            // Request account access  
            await window.ethereum.request({ method: 'eth\_requestAccounts' });

            // Instantiate Ethers provider and signer  
            const ethersProvider \= new ethers.BrowserProvider(window.ethereum);  
            const userSigner \= await ethersProvider.getSigner();  
            const userAccount \= await userSigner.getAddress();

            // Update global state  
            setProvider(ethersProvider);  
            setSigner(userSigner);  
            setAccount(userAccount);

        } catch (error) {  
            console.error("Error connecting to MetaMask:", error);  
        }  
    };  
      
    // Optional: Effect to handle account changes in MetaMask  
    useEffect(() \=\> {  
        if (window.ethereum) {  
            window.ethereum.on('accountsChanged', (accounts) \=\> {  
                if (accounts.length \> 0) {  
                    setAccount(accounts);  
                    // Re-initialize signer if account changes  
                    const ethersProvider \= new ethers.BrowserProvider(window.ethereum);  
                    ethersProvider.getSigner().then(setSigner);  
                } else {  
                    // User disconnected all accounts  
                    setProvider(null);  
                    setSigner(null);  
                    setAccount(null);  
                }  
            });  
        }  
    },);

    return (  
        \<div\>  
            {account? (  
                \<p\>Connected: {\`${account.substring(0, 6)}...${account.substring(account.length \- 4)}\`}\</p\>  
            ) : (  
                \<button onClick\={connectWallet}\>Connect Wallet\</button\>  
            )}  
        \</div\>  
    );  
};

export default ConnectWalletButton;

#### **Step-by-Step Logic Breakdown**

1. **Detecting MetaMask:** The connectWallet function first checks for the existence of window.ethereum.1 This object is injected into the browser by MetaMask and other EIP-1193 compliant wallets. If it's not present, the user is prompted to install MetaMask.  
2. **Requesting Accounts:** The line await window.ethereum.request({ method: 'eth\_requestAccounts' }); is the standardized way to trigger the MetaMask pop-up that asks the user for permission to connect their wallet to the site.1  
3. **Instantiating Ethers:** Once permission is granted, a new Ethers BrowserProvider is created with window.ethereum as its connection.14 This object represents a read-only connection to the blockchain via the node that MetaMask is currently connected to.  
4. **Getting the Signer:** The provider.getSigner() method is called to retrieve the Signer object.15 This object represents the user's active account and is required to sign and send any transaction that modifies the state of the blockchain (e.g., making a donation).  
5. **State Management:** The provider, signer, and the connected user's account address are stored in the global React Context, making them accessible to any other component in the application that needs to interact with the blockchain.

The Provider/Signer abstraction in Ethers.js is a powerful design pattern that directly maps to the read/write permission model of blockchain interaction. The Provider is a read-only interface; any component can use it to display on-chain data like project details or funding status. The Signer, however, is a privileged object that represents the user's ability to authorize state changes. By architecting the application to pass the Signer only to components that absolutely require it (like a "Donate" or "Verify" button), the application's attack surface is minimized. This ensures that only the necessary components have the power to request transaction signatures from the user, leading to a more secure and understandable codebase.

## **Part 4: The Unified Deployment and Configuration Strategy**

The final task for Day 2 is to deploy the completed smart contract suite to the Polygon Mumbai testnet. Because the system involves two interconnected contracts where one's address must be passed to the other's constructor, and ownership must then be transferred, a manual deployment process is tedious and highly prone to error. A Hardhat deployment script provides a repeatable, automated, and reliable solution.

### **Crafting the Hardhat Deployment Script**

A deployment script is a JavaScript or TypeScript file located in the scripts/ directory of a Hardhat project. It uses Ethers.js and the Hardhat Runtime Environment to programmatically deploy and configure contracts.

The following script automates the entire process for the ImpactChain system.16

**scripts/deploy.js**

JavaScript

const { ethers } \= require("hardhat");

async function main() {  
    const \[deployer\] \= await ethers.getSigners();  
    console.log("Deploying contracts with the account:", deployer.address);

    // 1\. Deploy the ImpactToken contract  
    const ImpactToken \= await ethers.getContractFactory("ImpactToken");  
    const impactToken \= await ImpactToken.deploy();  
    await impactToken.waitForDeployment(); // Use new syntax for Hardhat/Ethers v6  
    const impactTokenAddress \= await impactToken.getAddress();  
    console.log(\`ImpactToken deployed to: ${impactTokenAddress}\`);

    // 2\. Deploy the ProjectEscrow contract, passing the ImpactToken address to its constructor  
    const ProjectEscrow \= await ethers.getContractFactory("ProjectEscrow");  
    const projectEscrow \= await ProjectEscrow.deploy(impactTokenAddress);  
    await projectEscrow.waitForDeployment();  
    const projectEscrowAddress \= await projectEscrow.getAddress();  
    console.log(\`ProjectEscrow deployed to: ${projectEscrowAddress}\`);

    // 3\. Transfer ownership of the ImpactToken contract to the ProjectEscrow contract  
    console.log("Transferring ownership of ImpactToken to ProjectEscrow...");  
    const tx \= await impactToken.transferOwnership(projectEscrowAddress);  
    await tx.wait(); // Wait for the transaction to be mined  
    console.log("Ownership transferred successfully.");

    console.log("\\n--- Deployment Complete \---");  
    console.log(\`Verifier/Owner Address: ${deployer.address}\`);  
    console.log(\`ImpactToken Address: ${impactTokenAddress}\`);  
    console.log(\`ProjectEscrow Address: ${projectEscrowAddress}\`);  
    console.log("---------------------------\\n");  
}

main()  
   .then(() \=\> process.exit(0))  
   .catch((error) \=\> {  
        console.error(error);  
        process.exit(1);  
    });

#### **Step-by-Step Script Logic**

1. **Deploy $ImpactToken$**: The script first gets a contract factory for $ImpactToken$ and deploys it. It waits for the deployment transaction to be confirmed on the blockchain and then logs its new address.  
2. **Deploy $ProjectEscrow$**: Next, it deploys the $ProjectEscrow$ contract. Crucially, it passes the just-deployed impactTokenAddress as an argument to its constructor, establishing the permanent link between the two contracts.  
3. **Transfer Ownership**: This is the final and most critical step. The script calls the transferOwnership function on the deployed impactToken instance, passing the projectEscrowAddress as the new owner. This transaction programmatically establishes the "gated minting" mechanism.

This deployment script is more than a simple utility; it is the "genesis block" of the application's security model. The *sequence* of operations within the script is as important as the Solidity code itself. The final transferOwnership call is the single action that activates the entire security architecture. If this step were forgotten or executed incorrectly, the core promise of "cryptographic truth" would be compromised from the outset, as anyone could still call the safeMint function. Therefore, the deployment script must be treated as a piece of mission-critical infrastructure, version-controlled and recognized as the definitive setup procedure for the entire dApp.

### **Executing the Deployment and Verifying the Setup**

To run the script and deploy the contracts to the Polygon Mumbai testnet, execute the following command from the project's root directory:

Bash

npx hardhat run scripts/deploy.js \--network mumbai

After the script finishes, it will output the final deployed addresses. The setup can be externally verified using a block explorer like Polygonscan. Navigate to the deployed $ImpactToken$ contract's address on Polygonscan, select the "Contract" tab, and use the "Read Contract" functionality to query the owner() function. The result should be the address of the deployed $ProjectEscrow$ contract, providing objective proof that the system is configured correctly.

### **Exporting Artifacts for Frontend Consumption**

After compilation, Hardhat generates Application Binary Interface (ABI) files in the artifacts/ directory. The ABI is a JSON file that describes a contract's functions and is required by the frontend to interact with the contract. The deployed addresses and the ABIs must be made available to the frontend application. A simple manual step is to create a configuration file in the frontend source directory (e.g., frontend/src/contract-config.json) and copy the necessary information into it.

**contract-config.json Example**

JSON

{  
  "projectEscrowAddress": "0x...",  
  "impactTokenAddress": "0x...",  
  "projectEscrowAbi": \[... \],  
  "impactTokenAbi": \[... \]  
}

This file becomes the single source of truth for the frontend when connecting to the deployed smart contracts.

### **Table 1: Project Configuration and Deployed Artifacts Summary**

The following table serves as a centralized checklist and reference for all critical configuration parameters and post-deployment artifacts. Maintaining this summary ensures a smooth and error-free transition from backend deployment to frontend integration.

| Parameter | Value / Placeholder | Description |
| :---- | :---- | :---- |
| **Network** | Polygon Mumbai | The public testnet for deployment and demonstration. |
| **RPC Endpoint URL** | https://polygon-mumbai.infura.io/v3/YOUR\_API\_KEY | The connection URL to the blockchain node, configured in hardhat.config.js. |
| **Deployer Private Key** | 0x... (Stored in .env file) | The private key of the account used to deploy contracts and act as the "Verifier". Must be funded with testnet MATIC. |
| **$ImpactToken$ Address** | \[Output from deployment script\] | The final, deployed address of the NFT contract on the Mumbai testnet. |
| **$ProjectEscrow$ Address** | \[Output from deployment script\] | The final, deployed address of the main escrow contract on the Mumbai testnet. |
| **Contract ABIs** | frontend/src/contract-config.json | The path within the frontend project where the contract interfaces (ABIs) and addresses are stored for consumption by Ethers.js. |

## **Conclusion: A Deployed and Interactive Foundation for Day 3**

The successful completion of the Day 2 tasks marks a significant milestone in the hackathon sprint. The team now possesses a complete, interconnected, and deployed smart contract system residing on a public testnet. The entirety of the project's core on-chain logic—from project creation and donation to milestone verification, fund release, and final impact token minting—is now finalized and immutable.

Furthermore, the crucial bridge between this on-chain world and the user has been built. The frontend application has the foundational "plumbing" to communicate with the blockchain. With the ability to connect a user's wallet, the application is now aware of a user's on-chain identity and has the capacity to request transaction signatures. The project has evolved from a collection of abstract code into a live, interactive dApp skeleton. This robust and verified foundation enables the team to proceed to Day 3 with confidence, ready to focus entirely on building out the user-facing features of the donation flow, knowing that the underlying smart contract logic and connectivity are secure and operational.

#### **Works cited**

1. Hackathon Demo Roadmap\_ ImpactChain.pdf  
2. ERC 721 \- OpenZeppelin Docs, accessed September 13, 2025, [https://docs.openzeppelin.com/contracts/3.x/api/token/erc721](https://docs.openzeppelin.com/contracts/3.x/api/token/erc721)  
3. ERC 721 \- OpenZeppelin Docs, accessed September 13, 2025, [https://docs.openzeppelin.com/contracts/4.x/api/token/erc721](https://docs.openzeppelin.com/contracts/4.x/api/token/erc721)  
4. Access Control \- OpenZeppelin Docs, accessed September 13, 2025, [https://docs.openzeppelin.com/contracts/4.x/access-control](https://docs.openzeppelin.com/contracts/4.x/access-control)  
5. Ownership \- OpenZeppelin Docs, accessed September 13, 2025, [https://docs.openzeppelin.com/contracts/2.x/api/ownership](https://docs.openzeppelin.com/contracts/2.x/api/ownership)  
6. Access Control \- OpenZeppelin Docs, accessed September 13, 2025, [https://docs.openzeppelin.com/contracts/4.x/api/access](https://docs.openzeppelin.com/contracts/4.x/api/access)  
7. Access Control \- OpenZeppelin Docs, accessed September 13, 2025, [https://docs.openzeppelin.com/contracts/5.x/api/access](https://docs.openzeppelin.com/contracts/5.x/api/access)  
8. ERC721 \- OpenZeppelin Docs, accessed September 13, 2025, [https://docs.openzeppelin.com/contracts-cairo/0.11.0/erc721](https://docs.openzeppelin.com/contracts-cairo/0.11.0/erc721)  
9. ERC721 \- OpenZeppelin Docs, accessed September 13, 2025, [https://docs.openzeppelin.com/contracts/3.x/erc721](https://docs.openzeppelin.com/contracts/3.x/erc721)  
10. ERC721 \- OpenZeppelin Docs, accessed September 13, 2025, [https://docs.openzeppelin.com/contracts/2.x/erc721](https://docs.openzeppelin.com/contracts/2.x/erc721)  
11. Documentation \- Ethers.js, accessed September 13, 2025, [https://docs.ethers.org/v5/](https://docs.ethers.org/v5/)  
12. How to Connect Your Dapp With MetaMask Using Ethers.js | QuickNode Guides, accessed September 13, 2025, [https://www.quicknode.com/guides/ethereum-development/dapps/how-to-connect-your-dapp-with-metamask-using-ethersjs](https://www.quicknode.com/guides/ethereum-development/dapps/how-to-connect-your-dapp-with-metamask-using-ethersjs)  
13. Ethers.js, accessed September 13, 2025, [https://docs.ethers.org/v6/](https://docs.ethers.org/v6/)  
14. provides a connection to the blockchain, whch can be used to query its current state, simulate execution and send transactions to update the state., accessed September 13, 2025, [https://docs.ethers.org/v6/api/providers/](https://docs.ethers.org/v6/api/providers/)  
15. Getting Started \- Ethers.js, accessed September 13, 2025, [https://docs.ethers.org/v5/getting-started/](https://docs.ethers.org/v5/getting-started/)  
16. Deploying within Hardhat scripts | Ethereum development environment for professionals by Nomic Foundation, accessed September 13, 2025, [https://hardhat.org/ignition/docs/guides/scripts](https://hardhat.org/ignition/docs/guides/scripts)  
17. Deploy a Smart Contract Using Hardhat and Hiero JSON-RPC Relay \- Hedera Docs, accessed September 13, 2025, [https://docs.hedera.com/hedera/readme/tutorials/smart-contracts/deploy-a-smart-contract-using-hardhat-hedera-json-rpc-relay](https://docs.hedera.com/hedera/readme/tutorials/smart-contracts/deploy-a-smart-contract-using-hardhat-hedera-json-rpc-relay)  
18. How to Create and Deploy a Smart Contract with Hardhat | QuickNode Guides, accessed September 13, 2025, [https://www.quicknode.com/guides/ethereum-development/smart-contracts/how-to-create-and-deploy-a-smart-contract-with-hardhat](https://www.quicknode.com/guides/ethereum-development/smart-contracts/how-to-create-and-deploy-a-smart-contract-with-hardhat)