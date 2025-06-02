## Title
Lack of URL input sanitization in tokenize() function allows multiple tokenization of the same resources

#### validation
![Screenshot from 2025-06-02 13-18-48](https://github.com/user-attachments/assets/02a1cdee-f13b-4b3e-824a-81393752d1c2)


## Finding description
The tokenize function in UpsideProtocol.sol lacks URL input sanitization, allowing attackers to create multiple tokens for the same resource through various URL manipulation techniques.

## Vulnerability details
The tokenize function accepts any URL string without validation, which means it treats different representations of the same website as completely different resources. This allows an attacker to tokenize the same website multiple times by simply changing how the URL is written.

For example, these URLs all point to the same website (example.com), but the contract treats them as different resources:
https://example.com
https://EXAMPLE.com (different capitalization)
http://example.com (different protocol)
https://example.com/ (with trailing slash)
The contract also accepts:
Empty URLs ("")
Extremely long URLs (10,000+ characters)
URLs with special characters (!@#$%^&*)
URLs with Unicode characters that look like regular letters
URLs without a protocol (//example.com)
URLs with multiple subdomains (sub1.sub2.sub3.example.com)
URLs with path traversal attempts (../../../etc/passwd)

## Severity rational
The vulnerability is easily exploitable, on top of that the function is external which mean everyone can call it and no further input sanitization is realized nor upstream nor upstream.

## Recommended mitigation steps
1-Implement URL normalization before tokenization
2-Add URL validation checks

Idea for URL validation

```solidity
function validateURL(string memory url) internal pure returns (bool) {
    // Check for empty URL
    if (bytes(url).length == 0) return false;
    
    // Check URL length (prevent gas issues)
    if (bytes(url).length > 2048) return false;
    
    // Check for valid protocol (only https)
    if (!startsWith(url, "https://")) return false;
    
    return
```
Another idea to prevent duplicate tokenization.

```solidity
// Add to contract state
mapping(string => bool) private normalizedURLExists;

// Modify tokenize function
function tokenize(string memory url, address tokenAddress) external {
    // Normalize URL
    string memory normalizedUrl = normalizeURL(url);
    
    // Validate URL
    require(validateURL(normalizedUrl), "Invalid URL");
    
    // Check if URL already exists
    require(!normalizedURLExists[normalizedUrl], "URL already tokenized");
    
    // Mark URL as tokenized
    normalizedURLExists[normalizedUrl] = true;
    
    // Continue with tokenization...
}
```
URL validation is a complex and expensive mechanism in terms of both gas costs and implementation complexity. Before implementing any URL validation, consider:
Gas Optimization
Alternative Approaches like offchain validation
Trade-Off -> I understand the trade off between strict validation and gas costs but Tokenize() func at least needs to have input validation.

## Proof of Concept
The UpsideProtocolExploit.test.js test file demonstrates that the tokenize function accepts any URL string without validation, allowing the same resource to be tokenized multiple times through simple URL manipulations like case changes, protocol variations, and trailing slashes.

```typescript
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("UpsideProtocol Exploit Tests", function () {
    let UpsideProtocol;
    let protocol;
    let owner;
    let user1;
    let mockToken;
    let mockStaking;

    beforeEach(async function () {
        [owner, user1] = await ethers.getSigners();

        // Deploy mock ERC20 token for fees
        const MockToken = await ethers.getContractFactory("MockERC20");
        mockToken = await MockToken.deploy("Mock Token", "MTK", ethers.parseEther("1000000"));
        await mockToken.waitForDeployment();

        // Deploy mock staking contract
        const MockStaking = await ethers.getContractFactory("MockUpsideStaking");
        mockStaking = await MockStaking.deploy();
        await mockStaking.waitForDeployment();

        // Deploy UpsideProtocol
        UpsideProtocol = await ethers.getContractFactory("UpsideProtocol");
        protocol = await UpsideProtocol.deploy(owner.address);
        await protocol.waitForDeployment();

        // Initialize protocol
        await protocol.init(mockToken.target);
        await protocol.setStakingContractAddress(mockStaking.target);
    });

    describe("Tokenize Function Critical Vulnerabilities", function () {
        // Test 1: Empty URL - Could lead to tokenization of empty resources
        it("should allow tokenizing empty URL", async function () {
            const emptyUrl = "";
            await protocol.tokenize(emptyUrl, mockToken.target);
            // This could create a token for nothing, potentially causing market confusion
        });

        // Test 2: Extremely Long URL - Could cause gas issues or storage problems
        it("should allow tokenizing extremely long URL", async function () {
            const longUrl = "https://example.com/" + "a".repeat(10000); // 10KB URL
            await protocol.tokenize(longUrl, mockToken.target);
            // This could cause high gas costs and storage issues
        });

        // Test 3: Duplicate URL - Could lead to market manipulation
        it("should allow tokenizing same URL with different case", async function () {
            const url1 = "https://example.com";
            const url2 = "https://EXAMPLE.com";
            await protocol.tokenize(url1, mockToken.target);
            await protocol.tokenize(url2, mockToken.target);
            // This could be used to create multiple tokens for the same resource
        });

        // Test 4: Protocol Spoofing - Could lead to phishing or confusion
        it("should allow tokenizing same URL with different protocols", async function () {
            const url1 = "https://example.com";
            const url2 = "http://example.com";
            await protocol.tokenize(url1, mockToken.target);
            await protocol.tokenize(url2, mockToken.target);
            // This could be used for phishing attacks
        });

        // Test 5: Malicious Script Injection - Could lead to XSS if URL is used in web context
        it("should allow tokenizing URL with malicious script", async function () {
            const maliciousUrl = "https://example.com/<script>alert(document.cookie)</script>";
            await protocol.tokenize(maliciousUrl, mockToken.target);
            // If this URL is used in a web context, it could lead to XSS attacks
        });

        // Test 6: URL with Special Characters - Could lead to parsing issues
        it("should allow tokenizing URL with special characters", async function () {
            const specialCharsUrl = "https://example.com/!@#$%^&*()_+{}|:<>?";
            await protocol.tokenize(specialCharsUrl, mockToken.target);
            // This could cause issues with URL parsing and validation
        });

        // Test 7: Path Traversal - Could lead to unauthorized access if URL is used in file operations
        it("should allow tokenizing URL with path traversal", async function () {
            const pathTraversalUrl = "https://example.com/../../../etc/passwd";
            await protocol.tokenize(pathTraversalUrl, mockToken.target);
            // If this URL is used in file operations, it could lead to unauthorized access
        });

        // Test 8: Unicode Homograph Attack - Could lead to phishing
        it("should allow tokenizing URL with unicode homograph", async function () {
            const homographUrl = "https://exаmple.com"; // Using Cyrillic 'а' instead of Latin 'a'
            await protocol.tokenize(homographUrl, mockToken.target);
            // This could be used to create tokens that look legitimate but point to malicious sites
        });

        // Test 9: Protocol-relative URL - Could lead to protocol confusion
        it("should allow tokenizing protocol-relative URL", async function () {
            const protocolRelativeUrl = "//example.com";
            await protocol.tokenize(protocolRelativeUrl, mockToken.target);
            // This could lead to protocol confusion and security issues
        });

        // Test 10: URL with Multiple Subdomains - Could lead to domain confusion
        it("should allow tokenizing URL with multiple subdomains", async function () {
            const multiSubdomainUrl = "https://sub1.sub2.sub3.example.com";
            await protocol.tokenize(multiSubdomainUrl, mockToken.target);
            // This could be used to create tokens that look like they belong to different domains
        });
    });
});
```

Logs

```bash
npx hardhat test test/UpsideProtocolExploit.test.js


  UpsideProtocol Exploit Tests
    Tokenize Function Critical Vulnerabilities
      ✔ should allow tokenizing empty URL
      ✔ should allow tokenizing extremely long URL
      ✔ should allow tokenizing same URL with different case
      ✔ should allow tokenizing same URL with different protocols
      ✔ should allow tokenizing URL with malicious script
      ✔ should allow tokenizing URL with special characters
      ✔ should allow tokenizing URL with path traversal
      ✔ should allow tokenizing URL with unicode homograph
      ✔ should allow tokenizing protocol-relative URL
      ✔ should allow tokenizing URL with multiple subdomains


  10 passing (758ms)
Links to affected code
```
