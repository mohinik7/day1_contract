

# **ImpactChain Hackathon: A Developer's Implementation Guide to the Core Donation and Verification Loop**

## **Section 1: Architecting the Frontend for On-Chain Interaction**

A successful hackathon prototype is built not just on speed, but on a foundation of sound architectural decisions that enable that speed. Before writing a single line of UI code for the donation or verification flows, it is critical to establish robust, reusable patterns for managing blockchain state and interacting with the deployed smart contracts. This section outlines two foundational architectural components—a centralized contract service and a global state management context—that will significantly de-risk development, improve code clarity, and accelerate the implementation of the core features outlined for Day 3 and Day 4\.1

### **1.1 The Centralized Contract Service: A Single Source of Truth for Blockchain Logic**

The ImpactChain roadmap specifies a two-contract architecture on-chain ($ProjectEscrow.sol and $ImpactToken.sol) to ensure a clear separation of concerns.1 This best practice of modularity and single responsibility should be mirrored in the frontend architecture. A common anti-pattern, often seen in introductory tutorials, involves instantiating contract objects and making blockchain calls directly within UI components.2 This approach leads to scattered, duplicated, and difficult-to-maintain code, violating core software engineering principles.

A superior, professional-grade pattern is to create a dedicated service module that acts as the sole intermediary between the application's user interface and the smart contracts. This module encapsulates all the low-level logic of ethers.js, exposing a clean, high-level API to the rest of the application.

**Implementation Strategy:**

A new file, services/contractService.js, will be created. This module will be responsible for the following:

1. **Initialization:** It will import the Application Binary Interfaces (ABIs) and deployed contract addresses that were saved during the Day 2 deployment process.1  
2. **Contract Instantiation:** It will contain a function that takes a signer or provider object from ethers.js and returns instantiated Contract objects. This is the central point where new ethers.Contract(address, abi, signerOrProvider) is called.3 By centralizing this, the application avoids creating multiple, redundant contract instances across different components.  
3. **High-Level Function Exposure:** The service will export a set of asynchronous functions that map directly to the user's desired actions. These functions will handle the details of calling the correct contract methods and formatting the arguments. For example:  
   * getProjectDetails(projectId): Calls the projects(projectId) public getter on $ProjectEscrow.sol.  
   * getProjectCount(): Calls the projectCounter() public getter.  
   * donateToProject(projectId, amount): Calls the donate(projectId) function, correctly formatting the amount and passing it in the value field of the transaction overrides.  
   * verifyMilestone(projectId, milestoneIndex): Calls the verifyMilestone(projectId, milestoneIndex) function.  
   * getContractOwner(): Calls the owner() function inherited from Ownable.sol.

By adopting this centralized service pattern, the UI components become significantly simpler. They are no longer concerned with the intricacies of ethers.js; they simply call intuitive functions like await contractService.donateToProject(...) and handle the results. This decoupling makes the codebase easier to read, test, and debug under the tight constraints of a hackathon.

### **1.2 A Robust Web3 State Management Pattern with React Context**

A decentralized application's state is fundamentally tied to the user's wallet connection. Key pieces of information—such as the connection status, the user's address, and the ethers.js provider and signer objects—need to be accessible by many components throughout the application.1 Passing this information down through multiple layers of components via props, a practice known as "prop-drilling," is inefficient and leads to brittle, cluttered code.

The standard solution in a React application is to use the Context API to create a global state provider. This allows any component wrapped by the provider to access the shared Web3 state without intermediate props.

**Implementation Strategy:**

A Web3Context.js file will be created to encapsulate all wallet-related state and logic.

1. **Context Creation:** A new React Context is created to hold the Web3 state, including provider, signer, address, and isConnected.  
2. **Provider Component:** A Web3Provider component is built to wrap the entire application. This component will contain the core wallet connection logic.  
   * It will feature a connectWallet function that performs the necessary checks and requests:  
     * Verifies the presence of window.ethereum to detect a browser wallet like MetaMask.2  
     * Requests account access from the user by calling provider.send("eth\_requestAccounts",).3  
     * Upon successful connection, it instantiates an ethers.BrowserProvider (for ethers.js v6+) or ethers.providers.Web3Provider (for v5) with window.ethereum.4  
     * It retrieves the signer object, which is required to send transactions that modify the blockchain state.8  
   * The provider, signer, user address, and connection status are stored in the component's state and passed into the Context provider's value.  
3. **Custom Hook:** A custom hook, useWeb3(), is exported. This hook provides a clean and simple way for any child component to access the Web3 context with a single line of code, for example: const { signer, address, isConnected } \= useWeb3();.

This pattern ensures that wallet state is managed cleanly in one location, providing a stable and easily accessible foundation upon which the rest of the application's features can be built.

## **Section 2: Day 3 Implementation: Building the Donation Flow**

With the foundational architecture in place, Day 3 focuses on implementing the primary user journey: discovering a project and making a donation.1 This involves fetching on-chain data to populate the UI and, most critically, orchestrating a secure and user-friendly transaction flow.

### **2.1 Fetching and Displaying NGO Projects**

The first step in the user journey is to present the user with a list of vetted NGO projects available for funding. This data is not hardcoded; it resides on the Polygon Mumbai testnet within the deployed $ProjectEscrow smart contract and must be fetched dynamically.1

**Implementation in the NGOdiscovery.js Component:**

1. **Determine Project Count:** The component will first need to know how many projects exist. Using the contract service established in Section 1.1, it will make a read-only call to the projectCounter public state variable on the $ProjectEscrow contract. This function is automatically generated by the Solidity compiler and is gas-less.  
2. **Fetch Project Data:** The component will then iterate from 0 to projectCounter \- 1\. In each iteration, it will call another auto-generated public getter function, projects(i), where i is the project ID. This call returns the complete Project struct for that specific project, containing all its on-chain data (e.g., creator address, totalAmount, fundsRaised).  
3. **Render UI Cards:** For each Project struct retrieved, the component will render a UI card. This card will display key information to the user, such as the project's description (which can be mapped from an off-chain static JSON file using the project ID as a key, as suggested in the roadmap) and its funding progress. The progress can be visualized with a progress bar, calculated dynamically from the on-chain values: (fundsRaised / totalAmount) \* 100\. Each card will also feature a "Donate Now" button that navigates the user to the specific donation page for that project, passing the project ID as a URL parameter (e.g., /donate/0).

This process ensures that the discovery page is a true reflection of the on-chain state, directly demonstrating the transparency promised by the ImpactChain platform.

### **2.2 The Donation Transaction Lifecycle: A Masterclass in dApp UX**

Executing the donation is the most critical user interaction in the entire demo flow. The roadmap calls for a "polished and professional user interface" and an "intuitive and understandable" demonstration.1 A simple

await on a transaction call is insufficient, as it leaves the UI frozen and unresponsive, creating what can be called the "UX Chasm" of dApp development. A blockchain transaction is not an atomic, instantaneous event; it is a multi-stage process that must be clearly communicated to the user.

**Implementation in the DonationPage.js Component:**

The onClick handler for the "Donate Securely" button will orchestrate a carefully managed transaction lifecycle.

1. **Input Handling and Conversion:** The user will enter a donation amount in a human-readable format, such as "0.1" MATIC. Before this can be sent to the smart contract, it must be converted to its smallest denomination, WEI (1018 WEI \= 1 MATIC). This is a frequent source of errors for developers. The ethers.js library provides a utility function, ethers.parseEther("0.1"), to handle this conversion reliably.2  
2. **Transaction Construction:** The donation transaction will be initiated through the contract service. The call to the donate function must include an overrides object where the value property is set to the WEI amount calculated in the previous step. The code will look similar to: const tx \= await contract.donate(projectId, { value: donationInWei });.2  
3. **State Management and User Feedback:** To bridge the UX chasm, the component will use a local state variable to track the transaction's status (e.g., 'idle', 'pending', 'mining', 'success', 'error'). This state will drive conditional rendering of UI elements to provide continuous feedback. After the transaction is sent, the application will use await tx.wait(); to pause execution until the transaction has been included in a block and confirmed.2

The following table provides a clear blueprint for mapping the technical stages of an ethers.js transaction to a professional user experience.

| Transaction Phase | Code Trigger | UI State / Component | User-Facing Message |
| :---- | :---- | :---- | :---- |
| **Idle** | Component Mount | "Donate" button enabled | "Enter donation amount" |
| **Awaiting Confirmation** | contract.donate(...) is called | Modal appears, button disabled | "Please confirm the transaction in your wallet (e.g., MetaMask)." |
| **Mining / Processing** | tx promise resolves | Loading spinner in modal | "Your donation is being processed on the blockchain. Please wait..." |
| **Success** | await tx.wait() resolves | Success icon in modal, confetti | "Donation successful\! Thank you for your contribution." |
| **Error** | catch (error) block is hit | Error icon in modal | "Transaction failed. Please check the console for details." |

By meticulously implementing this lifecycle, the hackathon team can demonstrate not only functional on-chain logic but also a deep understanding of the nuances of Web3 user experience, a critical factor for judges evaluating the project's real-world viability.

## **Section 3: Day 4 Implementation: Closing the Loop with Verification and Impact Tokens**

Day 4 is about completing the core narrative of ImpactChain: transforming a donation into a verifiable, on-chain asset. This involves building the administrative interface for the Verifier role and implementing the logic to display the final Impact Token NFT, thereby "closing the loop" on the philanthropic journey.1

### **3.1 The Project Tracking Interface and Verifier's Action**

The platform's narrative requires a privileged "Verifier" role to approve project milestones, triggering the release of escrowed funds. The hackathon prototype cleverly simulates this complex role-based system using a simple on-chain surrogate: the owner of the smart contract.1 The frontend can leverage this on-chain data to create a powerful demonstration of access control without a traditional backend.

**Implementation in the ProjectTracking.js Component:**

1. **Fetch Detailed Project State:** When a user navigates to a specific project's tracking page (e.g., /project/0), the application will use the contract service to fetch the full Project struct, including the array of Milestone structs and their current states (Pending, Verified, or Paid). This data will be displayed clearly to the user.  
2. **On-Chain Access Control:** To implement the Verifier's functionality, the component will perform a simple but powerful check.  
   * Using the contract service, it will make a read-only call to the owner() function on the $ProjectEscrow contract. This function is inherited from OpenZeppelin's Ownable.sol contract and returns the address of the account that deployed it.1  
   * It will then retrieve the currently connected user's address from the Web3Context.  
   * A "Verify Milestone" button will be rendered next to each milestone that is in the Pending state. However, this button's visibility and interactivity will be conditional: it will only appear if the connected user's address matches the contract's owner address. This is a crucial step: if (connectedAddress.toLowerCase() \=== ownerAddress.toLowerCase()) { /\* render button \*/ }.  
3. **Executing Verification:** When the authorized verifier clicks the button, the frontend will trigger a transaction through the contract service, calling the verifyMilestone(\_projectId, \_milestoneIndex) function on the smart contract. This transaction will be managed using the same robust lifecycle pattern detailed in Section 2.2, providing clear feedback to the verifier. Upon successful confirmation, the application will automatically re-fetch the project data, and the UI will update to show the milestone's new state as "Paid," visually confirming the on-chain state change.

This implementation effectively demonstrates a core Web3 principle: using the blockchain itself as the application's state and permissioning machine. The public, on-chain owner() function acts as a definitive source of truth for user roles, enabling secure, decentralized access control directly within the frontend.

### **3.2 Displaying the Verifiable Impact Asset (The NFT)**

The final, culminating step of the ImpactChain loop is the issuance and display of the ImpactToken—an ERC-721 NFT that serves as an immutable, auditable record of the corporation's contribution.1 The frontend must be able to query the blockchain to find and display the NFTs belonging to the connected user.

**NFT Ownership Retrieval Strategy:**

A developer faces a key architectural choice when fetching a user's NFTs: querying the blockchain directly (on-chain enumeration) or using a specialized, indexed API service. For a hackathon, where performance and user experience are paramount, the choice is clear.

| Method | How it Works | Pros | Cons | Hackathon Recommendation |  
| :--- | :--- | :--- | :--- |  
| On-Chain Enumeration | Use the ERC721Enumerable extension. Call balanceOf(owner), then loop N times calling tokenOfOwnerByIndex(owner, i).14 | Fully decentralized; no third-party reliance. | Inefficient; requires N+1 separate network requests. Results in a slow, poor user experience. Higher gas cost for contract deployment.16 |  
Not Recommended. The severe performance penalty is unacceptable for a polished demo. |  
| Third-Party NFT API | Use a service like Alchemy or QuickNode. Make a single API call to an indexed endpoint (e.g., getNftsForOwner).17 | Extremely fast and efficient; a single API call retrieves all data. Simple to implement. Provides rich, pre-parsed metadata.17 | Introduces a centralization dependency. Requires a free API key. |  
**Strongly Recommended.** This approach aligns perfectly with the hackathon's goal of building a fast, visually compelling, and impressive demonstration under a tight deadline. |

**Implementation in the CorporateDashboard.js Component:**

1. **API Setup:** The team will sign up for a free account with a node provider like Alchemy and obtain an API key. The Alchemy SDK will be installed as a project dependency.20  
2. **Fetch User's NFTs:** A new function, fetchMyImpactTokens, will be created. This function will use the Alchemy SDK's alchemy.nft.getNftsForOwner method. It will pass the connected user's address and, crucially, use the contractAddresses filter option to request only the NFTs from the deployed $ImpactToken contract address.19 This ensures the query is targeted and efficient.  
3. **Retrieve and Render Metadata:** The application will iterate over the array of NFTs returned by the API. For each NFT, it will:  
   * Retrieve the tokenURI from the NFT's metadata. As specified in the roadmap, this URI will point to a JSON file hosted on a service like JSONBin.io or a GitHub Gist.1  
   * Fetch this JSON file. The file's content will adhere to the ERC-721 metadata standard, containing properties like name, description, image, and custom attributes that detail the specific social impact achieved.1  
   * Use this fetched data to render a visually appealing "Impact Certificate" component within a "My Impact Tokens" section of the dashboard. This provides the donor with tangible, cryptographic proof of their contribution, successfully closing the demonstration loop.

## **Section 4: Conclusion: From Code to Compelling Narrative**

The successful implementation of the features outlined for Day 3 and Day 4 transforms the ImpactChain concept from a slide deck into a tangible, functional prototype. By following this guide, the development team can construct a complete, end-to-end user journey that powerfully communicates the project's core value proposition.

The resulting demo will showcase a seamless flow: a corporation discovers a project, donates funds that are transparently locked in an on-chain escrow, a verifier approves a milestone triggering an automated fund release, and finally, the corporation receives a unique, non-fungible Impact Token as a verifiable asset. This is not merely a sequence of technical tasks; it is the enactment of the central narrative—the creation of "cryptographic truth in philanthropy".1 The architectural patterns and UX considerations detailed herein are designed not just to build a working application, but to build one that is polished, intuitive, and compelling, ensuring that the judges see not just code, but a clear and powerful vision for the future of corporate social responsibility.

#### **Works cited**

1. Hackathon Demo Roadmap\_ ImpactChain.pdf  
2. Steps to build a Web3 application with React, Vite, and Ethers.js \- Opcito, accessed September 13, 2025, [https://www.opcito.com/blogs/steps-to-build-a-web3-application-with-react-vite-and-ethersjs](https://www.opcito.com/blogs/steps-to-build-a-web3-application-with-react-vite-and-ethersjs)  
3. Building a React Frontend with Ethers.js | by Bishal Devkota | Medium, accessed September 13, 2025, [https://medium.com/@bishalf98/building-a-react-frontend-with-ethers-js-8d806459a4b0](https://medium.com/@bishalf98/building-a-react-frontend-with-ethers-js-8d806459a4b0)  
4. Interacting with Smart Contracts using ethers.js — ModeNetwork | by McTrick \- Medium, accessed September 13, 2025, [https://mctrick.medium.com/interacting-with-smart-contracts-using-ethers-js-modenetwork-5f59a41ea9de](https://mctrick.medium.com/interacting-with-smart-contracts-using-ethers-js-modenetwork-5f59a41ea9de)  
5. How to Get Started Using Ethers.js With React – Programmable ..., accessed September 13, 2025, [https://programmablewealth.com/ethersjs-react-tutorial/](https://programmablewealth.com/ethersjs-react-tutorial/)  
6. Contract \- Ethers.js, accessed September 13, 2025, [https://docs.ethers.org/v5/api/contract/contract/](https://docs.ethers.org/v5/api/contract/contract/)  
7. How to Mint an NFT on Polygon with Ethers.js | QuickNode Guides, accessed September 13, 2025, [https://www.quicknode.com/guides/other-chains/polygon/how-to-mint-an-nft-on-polygon-with-ethersjs](https://www.quicknode.com/guides/other-chains/polygon/how-to-mint-an-nft-on-polygon-with-ethersjs)  
8. React.js & Ethers.js: Demo of using the InfuraProvider vs the MetaMask wallet Web3Provider, accessed September 13, 2025, [https://support.metamask.io/develop/building-with-infura/javascript-typescript/infuraprovider-metamaskwalletprovider-react-ethersjs/](https://support.metamask.io/develop/building-with-infura/javascript-typescript/infuraprovider-metamaskwalletprovider-react-ethersjs/)  
9. How to force the transfer of an NFT using Web3.js or Ethers.js \- Stack Overflow, accessed September 13, 2025, [https://stackoverflow.com/questions/71894376/how-to-force-the-transfer-of-an-nft-using-web3-js-or-ethers-js](https://stackoverflow.com/questions/71894376/how-to-force-the-transfer-of-an-nft-using-web3-js-or-ethers-js)  
10. How to Send a Transaction Using Ethers.js | QuickNode Guides, accessed September 13, 2025, [https://www.quicknode.com/guides/ethereum-development/transactions/how-to-send-a-transaction-in-ethersjs](https://www.quicknode.com/guides/ethereum-development/transactions/how-to-send-a-transaction-in-ethersjs)  
11. Use ethers.js | MetaMask developer documentation, accessed September 13, 2025, [https://docs.metamask.io/services/tutorials/ethereum/send-a-transaction/send-a-transaction-ethers/](https://docs.metamask.io/services/tutorials/ethereum/send-a-transaction/send-a-transaction-ethers/)  
12. How to call a contract function/method using ethersjs \- Ethereum Stack Exchange, accessed September 13, 2025, [https://ethereum.stackexchange.com/questions/120817/how-to-call-a-contract-function-method-using-ethersjs](https://ethereum.stackexchange.com/questions/120817/how-to-call-a-contract-function-method-using-ethersjs)  
13. How to send a transaction using ethers.js v6 \- YouTube, accessed September 13, 2025, [https://www.youtube.com/shorts/cCpoXejzgsA](https://www.youtube.com/shorts/cCpoXejzgsA)  
14. How ERC721 Enumerable Works | By RareSkills, accessed September 13, 2025, [https://rareskills.io/post/erc-721-enumerable](https://rareskills.io/post/erc-721-enumerable)  
15. Where is "tokenIDForOwnerByIndex" ? · Issue \#2563 · OpenZeppelin/openzeppelin-contracts \- GitHub, accessed September 13, 2025, [https://github.com/OpenZeppelin/openzeppelin-contracts/issues/2563](https://github.com/OpenZeppelin/openzeppelin-contracts/issues/2563)  
16. How to retrieve all ERC721 tokens of a certain collection what a wallet ownes?, accessed September 13, 2025, [https://ethereum.stackexchange.com/questions/152051/how-to-retrieve-all-erc721-tokens-of-a-certain-collection-what-a-wallet-ownes](https://ethereum.stackexchange.com/questions/152051/how-to-retrieve-all-erc721-tokens-of-a-certain-collection-what-a-wallet-ownes)  
17. Look Up All NFTs for a Given Address on Ethereum Mainnet ..., accessed September 13, 2025, [https://www.quicknode.com/guides/quicknode-products/apis/look-up-all-nfts-for-a-given-address-on-ethereum-mainnet](https://www.quicknode.com/guides/quicknode-products/apis/look-up-all-nfts-for-a-given-address-on-ethereum-mainnet)  
18. NFT API Tutorials | Alchemy Docs, accessed September 13, 2025, [https://www.alchemy.com/docs/nft-api](https://www.alchemy.com/docs/nft-api)  
19. How to Get All NFTs Owned by an Address | Alchemy Docs, accessed September 13, 2025, [https://www.alchemy.com/docs/how-to-get-all-nfts-owned-by-an-address](https://www.alchemy.com/docs/how-to-get-all-nfts-owned-by-an-address)  
20. NFT API Quickstart | Alchemy Docs, accessed September 13, 2025, [https://www.alchemy.com/docs/reference/nft-api-quickstart](https://www.alchemy.com/docs/reference/nft-api-quickstart)  
21. Exploring the Alchemy NFT API \- DEV Community, accessed September 13, 2025, [https://dev.to/envoy\_/the-ultimate-nft-api-24d5](https://dev.to/envoy_/the-ultimate-nft-api-24d5)  
22. How to Create an NFT | Alchemy \- Medium, accessed September 13, 2025, [https://medium.com/alchemy-api/how-to-write-deploy-an-nft-d92c8b6b777f](https://medium.com/alchemy-api/how-to-write-deploy-an-nft-d92c8b6b777f)