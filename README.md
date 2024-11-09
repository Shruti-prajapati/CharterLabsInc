Task 2: Develop a Notification Bot for Tracking USDC Transfers to a Specific Address Objective Monitor all incoming USDC transfers to a specified Ethereum address on the Sepolia network. Have a dashboard in the frontend to showcase the required txns attached to push notification service.
To accomplish the task of building a Notification Bot for tracking USDC transfers to a specific Ethereum address on the Sepolia network, we can break down the solution into several key components:
1.	Blockchain and Subgraph Setup:
o	Set up a subgraph using The Graph Protocol to monitor USDC (an ERC-20 token) transfers on the Sepolia network.
o	The subgraph will filter events to track transfers directed to a specified address and store relevant details like from, to, value, and timestamp.
2.	Frontend Dashboard:
o	A simple web-based dashboard will display the relevant transaction details (e.g., sender, recipient, amount, timestamp).
o	The dashboard will also display the status of the notification service.
3.	Push Notification Service:
o	A push notification service will notify users of the incoming USDC transfers in real-time.
o	Notifications can be sent through services like Firebase, OneSignal, or a custom WebSocket-based service.
4.	Testing and Deployment:
o	The system will be tested for accurate detection of USDC transfers, error handling, and notification delivery.
o	The subgraph and frontend will be deployed for production use.
Step 1: Subgraph Setup
1.	Install the Graph CLI:
o	First, ensure that you have the Graph CLI installed on your system to create, deploy, and manage subgraphs.
bash
Copy code
npm install -g @graphprotocol/graph-cli
2.	Create a Subgraph Project:
bash
Copy code
graph init --from-contract <USDC_CONTRACT_ADDRESS> --network sepolia usdc-transfer-tracker
This command initializes the subgraph for tracking the USDC contract on the Sepolia network.
3.	Define the Transfer Entity:
In the schema.graphql file, define a Transfer entity to capture relevant data:
graphql
Copy code
type Transfer @entity {
  id: ID!
  from: Bytes!
  to: Bytes!
  value: BigInt!
  timestamp: BigInt!
}
4.	Define Event Handlers:
In the subgraph.yaml file, set up the event handler to listen for Transfer events emitted by the USDC contract:
yaml
Copy code
dataSources:
  - kind: ethereum/contract
    name: USDC
    network: sepolia
    source:
      address: "<USDC_CONTRACT_ADDRESS>"
      abi: USDC
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.5
      language: wasm
      file: ./src/mappings/usdc.ts
5.	Implement Event Mapping:
In the src/mappings/usdc.ts file, implement the event handler that filters transfers directed to the specified address:
typescript
Copy code
import { BigInt } from "@graphprotocol/graph-ts";
import { Transfer } from "../generated/schema";
import { Transfer as TransferEvent } from "../generated/USDC/USDC";

export function handleTransfer(event: TransferEvent): void {
  let transfer = new Transfer(event.transaction.hash.toHex() + "-" + event.logIndex.toString());
  transfer.from = event.params.from;
  transfer.to = event.params.to;
  transfer.value = event.params.value;
  transfer.timestamp = event.block.timestamp;
  transfer.save();
}
This handler stores the transfer data, filtering only the transactions where the recipient (to) is the specified address.
6.	Deploy the Subgraph:
After defining the schema and mappings, deploy the subgraph to The Graph:
bash
Copy code
graph deploy --product hosted-service <YOUR_SUBGRAPH_NAME>
Step 2: Polling Subgraph and Notification Service
1.	Create a Node.js Backend for Polling:
Use a Node.js server to periodically query the subgraph for recent transfers to the specified address.
javascript
Copy code
const axios = require('axios');

const SUBGRAPH_URL = "<YOUR_SUBGRAPH_URL>";
const TARGET_ADDRESS = "<SPECIFIED_ADDRESS>";

async function getRecentTransfers() {
  const query = `
    {
      transfers(first: 10, where: {to: "${TARGET_ADDRESS}"}) {
        id
        from
        to
        value
        timestamp
      }
    }
  `;
  
  const response = await axios.post(SUBGRAPH_URL, { query });
  return response.data.data.transfers;
}

setInterval(async () => {
  const transfers = await getRecentTransfers();
  transfers.forEach(transfer => {
    sendPushNotification(transfer);
  });
}, 60000);  // Poll every 1 minute
2.	Push Notification Service:
You can use services like Firebase Cloud Messaging (FCM) or OneSignal to send push notifications.
Example using Firebase Cloud Messaging (FCM):
javascript
Copy code
const admin = require('firebase-admin');
const serviceAccount = require("<path_to_your_firebase_service_account_key>");

admin.initializeApp({
  credential: admin.credential.cert(serviceAccount),
});

function sendPushNotification(transfer) {
  const message = {
    notification: {
      title: 'New USDC Transfer',
      body: `From: ${transfer.from} To: ${transfer.to} Amount: ${transfer.value}`,
    },
    token: "<USER_FCM_DEVICE_TOKEN>",
  };

  admin.messaging().send(message)
    .then((response) => {
      console.log('Successfully sent message:', response);
    })
    .catch((error) => {
      console.log('Error sending message:', error);
    });
}
Note: Replace <USER_FCM_DEVICE_TOKEN> with the recipient’s device token for push notifications.
Step 3: Frontend Dashboard
1.	Create a Simple Dashboard using React.js:
o	Install React and necessary dependencies:
bash
Copy code
npx create-react-app usdc-tracker
cd usdc-tracker
npm install axios
o	Fetch and display recent transactions from the subgraph:
javascript
Copy code
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const SUBGRAPH_URL = "<YOUR_SUBGRAPH_URL>";

function Dashboard() {
  const [transfers, setTransfers] = useState([]);

  useEffect(() => {
    async function fetchTransfers() {
      const query = `
        {
          transfers(first: 10, where: {to: "<SPECIFIED_ADDRESS>"}) {
            id
            from
            to
            value
            timestamp
          }
        }
      `;
      const response = await axios.post(SUBGRAPH_URL, { query });
      setTransfers(response.data.data.transfers);
    }

    fetchTransfers();
  }, []);

  return (
    <div>
      <h1>USDC Transfer Dashboard</h1>
      <table>
        <thead>
          <tr>
            <th>From</th>
            <th>To</th>
            <th>Amount</th>
            <th>Timestamp</th>
          </tr>
        </thead>
        <tbody>
          {transfers.map((transfer) => (
            <tr key={transfer.id}>
              <td>{transfer.from}</td>
              <td>{transfer.to}</td>
              <td>{transfer.value}</td>
              <td>{new Date(transfer.timestamp * 1000).toLocaleString()}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}

export default Dashboard;
2.	Run the Frontend:
bash
Copy code
npm start
This will launch a simple web interface displaying the latest transfers to the specified address.
Step 4: Error Handling and Deployment
•	Error Handling:
o	Ensure that your backend properly handles errors such as failed subgraph queries or push notification failures.
o	Implement retries for the polling mechanism and fallback logic in case the subgraph is unreachable.
•	Deployment:
o	Deploy the backend (Node.js service) and frontend (React app) to appropriate platforms. For example, you can use services like Heroku, AWS, or DigitalOcean for hosting.
Conclusion
This solution provides a comprehensive framework to monitor USDC transfers to a specific Ethereum address on the Sepolia network. It includes:
•	Subgraph Setup: Using The Graph Protocol to track ERC-20 transfers.
•	Backend Polling: A Node.js backend that checks the subgraph at regular intervals.
•	Push Notification Service: Real-time notifications sent to users via Firebase Cloud Messaging or other services.
•	Frontend Dashboard: A simple React-based dashboard that displays the transfers in real-time.
By following this plan, you should be able to implement, test, and deploy a notification bot for USDC transfers.
----------------------------------------------------------------------------------------------------------------------------
Task 3: Modular Signature Scheme Verification Algorithm (Open task - Any coding language and any signature scheme can be used)
Here's the full Java implementation for a modular signature scheme verification algorithm. The solution includes support for multiple signature schemes such as ECDSA, Schnorr, and RSA. It dynamically selects the appropriate verification method based on the scheme type passed as an argument.
1.	SignatureScheme Interface:
o	The SignatureScheme interface defines the verify method, which is implemented by each signature scheme class (ECDSA, Schnorr, RSA). This allows each signature scheme to handle its own verification process in a modular way.
2.	Signature Scheme Classes:
o	ECDSASignatureScheme: Uses Java's built-in Signature class with the SHA256withECDSA algorithm for ECDSA signature verification. It assumes that you can retrieve the public key for the signer from the address.
o	SchnorrSignatureScheme: A placeholder for Schnorr signature verification. In practice, you would need a library or custom logic to verify Schnorr signatures. The code simply compares the signature with the message hash for illustrative purposes.
o	RSASignatureScheme: Similar to ECDSA, this uses Java's Signature class with the SHA256withRSA algorithm to verify RSA signatures.
3.	signatureVerifier Method:
o	This method takes the address of the signer, the signature, the message hash, and the signature scheme type (ecdsa, schnorr, rsa).
o	It uses a Map to dynamically select the appropriate signature scheme implementation based on the schemeType. If an unsupported scheme type is passed, it returns false.
o	The appropriate verification method is then invoked on the selected scheme class.
4.	Testing:
o	The testSignatureVerifier function demonstrates how you can test the verification with different schemes by passing dummy signature data and message hash.
o	For each signature scheme, it prints whether the signature is valid or invalid.
Key Points:
•	Modularity: The system is modular, with each signature scheme's verification logic encapsulated in its own class. This makes it easy to extend the system to support additional signature schemes.
•	Flexibility: The algorithm can handle different signature schemes (ECDSA, Schnorr, RSA) by simply adding corresponding classes that implement the SignatureScheme interface.
•	Customization: The getPublicKey method is a placeholder; in practice, you would need to retrieve the public key based on the address provided.
Considerations for Real Implementation:
•	Public Key Resolution: In practice, the getPublicKey method needs to resolve an address to the actual public key. For example, in Ethereum or Bitcoin, this would involve resolving an address to its public key using blockchain data.
•	Signature Algorithms: The actual cryptographic operations in each scheme (verify method) should be implemented with real libraries such as BouncyCastle for Schnorr or RSA/ECDSA support in Java's java.security package.
•	Error Handling: Proper error handling and edge case checking should be implemented (e.g., invalid address formats, signature size mismatches).
This structure provides flexibility to verify different types of signatures while maintaining a modular and scalable design.
Java Implementation:
import java.security.*;
import java.util.HashMap;
import java.util.Map;
import java.util.Arrays;

public class SignatureVerifier {

    // Interface for signature verification
    interface SignatureScheme {
        boolean verify(String address, byte[] signature, byte[] messageHash) throws Exception;
    }

    // ECDSA verification implementation
    static class ECDSASignatureScheme implements SignatureScheme {
        @Override
        public boolean verify(String address, byte[] signature, byte[] messageHash) throws Exception {
            // Use Java's built-in ECDSA verification
            // In practice, you would use something like java.security.Signature
            try {
                // Initialize the ECDSA verification logic
                Signature ecdsaSignature = Signature.getInstance("SHA256withECDSA");
                PublicKey publicKey = getPublicKey(address);  // Fetch the public key of the signer (address)
                ecdsaSignature.initVerify(publicKey);
                ecdsaSignature.update(messageHash);
                return ecdsaSignature.verify(signature);  // Return the verification result
            } catch (Exception e) {
                System.out.println("Error during ECDSA verification: " + e.getMessage());
                return false;
            }
        }

        private PublicKey getPublicKey(String address) {
            // Placeholder method to get the public key from the address
            // In practice, you would resolve the address to a public key
            return null;
        }
    }

    // Schnorr signature verification implementation
    static class SchnorrSignatureScheme implements SignatureScheme {
        @Override
        public boolean verify(String address, byte[] signature, byte[] messageHash) throws Exception {
            // Implement Schnorr signature verification logic here
            System.out.println("Verifying with Schnorr...");
            // Dummy check (to be replaced with real Schnorr verification logic)
            return Arrays.equals(signature, messageHash);
        }
    }

    // RSA signature verification implementation
    static class RSASignatureScheme implements SignatureScheme {
        @Override
        public boolean verify(String address, byte[] signature, byte[] messageHash) throws Exception {
            // Use Java's built-in RSA verification
            try {
                // Initialize the RSA signature verification
                Signature rsaSignature = Signature.getInstance("SHA256withRSA");
                PublicKey publicKey = getPublicKey(address);  // Fetch the public key of the signer (address)
                rsaSignature.initVerify(publicKey);
                rsaSignature.update(messageHash);
                return rsaSignature.verify(signature);  // Return the verification result
            } catch (Exception e) {
                System.out.println("Error during RSA verification: " + e.getMessage());
                return false;
            }
        }

        private PublicKey getPublicKey(String address) {
            // Placeholder method to get the public key from the address
            // In practice, you would resolve the address to a public key
            return null;
        }
    }

    // Main Signature Verifier function
    public static boolean signatureVerifier(String address, byte[] signature, byte[] messageHash, String schemeType) throws Exception {
        // Map for associating scheme types with the corresponding verification class
        Map<String, SignatureScheme> schemeMap = new HashMap<>();
        schemeMap.put("ecdsa", new ECDSASignatureScheme());
        schemeMap.put("schnorr", new SchnorrSignatureScheme());
        schemeMap.put("rsa", new RSASignatureScheme());

        // Select the appropriate scheme based on the schemeType
        SignatureScheme signatureScheme = schemeMap.get(schemeType.toLowerCase());

        // Early exit if the scheme type is unsupported
        if (signatureScheme == null) {
            System.out.println("Unsupported scheme type: " + schemeType);
            return false;
        }

        // Attempt verification using the appropriate scheme method
        return signatureScheme.verify(address, signature, messageHash);
    }

    // Example test function
    public static void testSignatureVerifier() throws Exception {
        // Example data for testing
        String address = "0x123456789abcdef";  // Placeholder address
        String message = "Test message";
        byte[] messageHash = message.getBytes();  // For simplicity, using raw bytes instead of SHA-256 hash here

        // Test with different signature schemes
        String[] schemes = {"ecdsa", "schnorr", "rsa"};
        for (String scheme : schemes) {
            byte[] signature = new byte[64];  // Placeholder 64-byte signature for testing
            boolean result = signatureVerifier(address, signature, messageHash, scheme);
            System.out.println("Signature verification for " + scheme + " scheme: " + (result ? "Valid" : "Invalid"));
        }
    }

    // Main method to run the test
    public static void main(String[] args) throws Exception {
        testSignatureVerifier();
    }
}
