### 1 - collecting the files from PillarDAO
As you can see, PillarDAO is making use of 18 solidity files that we need to download to compile the project and start testing it.

![etherscan-pillarDAO-files](https://github.com/user-attachments/assets/1ff236ec-e93a-49c7-b328-c3b08f1f1f97)

An easy way to download bulk file is to click on the IDE button on the top right corner. Then you will end up in VS online on your web browser, just click right on the folder and click download, then import it your favorite IDE and start writting smoke tests !

### 2 - Verify if you're using the correct compiler version
in your foundry.toml, check the compiler version that it matches with settings.json provided. (settings.json is about hardhat but I'm using foundry.toml because I'm testing with forge and anvil)

[profile.default]
src = "contracts"
out = "out"
libs = ["lib"]
remappings = [
    "forge-std/=lib/forge-std/src/",
    "@openzeppelin/=lib/openzeppelin-contracts/"
] 
solc = "0.8.18" 
optimizer = true
optimizer_runs = 10000

### 3 - Once your smoke tests work you can try use slither

slither src/MembershipNFT.sol --foundry-ignore-compile
-> You might have error regarding compilation,  if  you double checks that it's not relevant you can skip them using --foundry-ignore-compile flag.

### 4 - Quickly understanding what is doing what:
PillarDAO.sol -> The main DAO contract <br>
MembershipNFT.sol ->The NFT contract for membership tokens <br>

This appears to be a long-term commitment DAO where:<br><br>
Members must make a financial commitment to join<br>
The one-year lock period ensures members are invested in the DAO's success<br>
The NFT provides a verifiable proof of membership<br>
The system could be used for governance, exclusive access, or shared benefits<br>
The combination of staking and NFT membership creates a dual verification system where both token commitment and membership status can be verified independently.<br>


Happy hacking.
