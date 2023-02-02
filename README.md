# How-To-Implement-The-Diamond-Standard
The Diamond Standard, EIP-2535, was created by Nick Mudge as a standardized architecture for implementing proxy smart contracts. In this article we will go over the pros of using the Diamond Standard and how it works.

## General Overview
The Diamond Standard operates by deploying a contract called ```Diamond.sol```. ```Diamond.sol```, then calls other smart contracts via ```delegatecall()```. This allows ```Diamond.sol``` to execute the code of the called contract in the context of ```Diamond.sol```. All contracts that are called with ```delegatecall()``` are referred to as facets. Facets can be replaced, removed, and added as time goes on, which allows developers to create modular apps on EVM compatible blockchains. The Diamond Standard requires you to implement  ```DiamondLoupeFacet.sol```. ```DiamondLoupeFacet.sol``` is in charge of documenting other facets in your protocol. ```DiamondLoupeFacet.sol``` provides transparency by allowing  users to see what the facet addresses and functions are. Another requirement of the Diamond Standard is ```DiamondCutFacet.sol```. ```DiamondCutFacet.sol``` is responsible for all upgrades to your application. ```DiamondCutFacet.sol``` also emits events when called, providing another level of transparency for users of the protocol. Although not necessarily a requirement, ```LibDiamond.sol``` is a library that provides many helper functions to write an application with the Diamond Standard. When creating an application with the Diamond Standard, you can go with one of two routes to maintain your state variables. They are referred to as ```App Storage``` and ```Diamond Storage```. We will go over all of the Diamond Standard’s components in more detail in the rest of the article.

The Diamond Standard has three different versions of implementation. Luckily, if you decide that you chose the wrong implementation, it is possible to upgrade your application to a different implementation. ```diamond-1``` is the most basic implementation. The complexity of ```diamond-1``` is the easiest to understand, and its gas costs are the most mild. It is not recommended to call ```DiamondLoupeFacet.sol```’s functions on chain with ```diamond-1``` (or ```diamond-2```), due to the intensive gas costs. ```diamond-3```, on the other hand, chooses to optimize calling ```DiamondLoupeFacet.sol```’s functions on chain. The trade off is that it is more costly to call ```DiamondCutFacet.sol```. ```diamond-2``` is very similar to ```diamond-1```, but optimizes gas costs over complexity to understand. It is up to you, as a developer, to decide which one of these implementations is the best fit for your application.

Due to the complexity of the Diamond Standard, it is recommended to follow a general template when creating your applications. I will link these templates at the bottom of the article along with further reading materials.

## Why Use The Diamond Standard?
The main criticism of the Diamond Standard is its complexity (hopefully I can help solve this). Although it is complex, the Diamond Standard provides the following benefits: <br>
- There is no practical limit to the size of your application. All smart contracts have a maximum size of 24KB. Since the Diamond Standard uses ```Diamond.sol``` to ```delegatecall()``` its facets, the size of the contract remains relatively small. <br>
- You only use one address for a multitude of functionality. Again, thanks to ```delegatecall()```, you can add all the functionality you want to a single smart contract. This includes implementing tokens with the same address as your protocol. <br>
- It provides an organized way to upgrade your smart contracts. This can help auditors to understand and protect your application. <br>
- It allows you to upgrade your smart contracts incrementally. Instead of having to redeploy your entire contract, you can simply add, replace, or remove a facet to provide the desired functionality to your application. <br>
- It provides a great level of transparency for your proxy contract. As previously mentioned, ```DiamondLoupeFacet.sol``` allows users to see where your functions are located on the blockchain and what your functions execute. Check out https://louper.dev/ to analyze the functionality of smart contracts that take advantage of the Diamond Standard. <br>
- There is nearly no limit to the amount of facets that can be utilized in your application. The only way that you would not be able to add more facets to your protocol would be if ```Diamond.sol``` ran out of space to store the facet data. This would need a ridiculously large amount of facets, to the point it is practically impossible. <br>
- Facets can be reused by multiple ```Diamond.sol``` contracts. This allows you to save gas costs when deploying applications that borrow functionality. <br>
- You can run the optimizer at a higher runs setting if desired. The Solidity optimizer helps to optimize the gas costs when calling external functions. However, it increases the contract deployment bytecode. This can cause your smart contract to go over the max smart contract size limit. Since you can deploy as many facets as you want, you can set the optimizer to as high of a number of runs you desire without having to worry about your contract being too large. <br>

There are many reasons why you would want to use the Diamond Standard in your application, but remember not to over complicate your application. If your protocol can work with a single smart contract and none of the above benefits apply to your application there is no need to use the Diamond Standard. It will just end up adding more complexity and gas costs to your application. However, if you are in the situation where you need a proxy pattern for your application, I personally recommend using the Diamond Standard.



## App Storage VS Diamond Storage
Architecture of state variables is one of the most important aspects of your blockchain application. I decided to cover this section first to give you a good understanding of how state variables are managed with the Diamond Standard. Obviously proxies heavily rely on ```delegatecall()``` to execute the code from your facet contracts in the context of your main contract (```Diamond.sol```). Since all of the storage of our state variables is kept in ```Diamond.sol```, we need to make sure our variables are not overwriting each other. Before we look at the correct way to organize our state variables, let’s look at an example of the wrong way to manage state variables.
```
pragma solidity^0.8.17;

contract Main {

    uint256 public verySpecialVar;
    uint256 public notSoSpecialVar;

    // delegate calls SpecialVarManager to update verySpecialVar
    function setVerySpecialVar(address _specialVarManager) external {
        _specialVarManager.delegatecall(
            abi.encodeWithSignature("writeSpecialVar()")
        );
    }

    // delegate calls NotSpecialVarManager to update notSoSpecialVar
    function setNotSoSpecialVar(address _notSpecialVarManager) external {
        _notSpecialVarManager.delegatecall(
            abi.encodeWithSignature("writeNotSpecialVar()")
        );
    }
    
}

contract SpecialVarManager {
    uint256 verySpecialVar;

    function writeSpecialVar() external {
        verySpecialVar = 100;
    }
}

contract NotSpecialVarManager {
    uint256 notSoSpecialVar;

    function writeNotSpecialVar() external {
        notSoSpecialVar = 50;
    }
}
```
If you call ```setVerySpecialVar()``` you see ```verySpecialVar``` has been updated to 100! Now let’s call ```setNotSoSpecialVar``` and see what happens. We see ```notSoSpecialVar``` is still not initialized and equal to 0. If we check ```verySpecialVar``` we see it's set to 50 now. Why? Well the storage layout in ```NotSpecialVarManager``` doesn't have ```verySpecialVar```. So when we call ```writeNotSpecialVar()``` via ```delegatecall()``` we are telling ```Main``` to update storage slot 0 to 50. Solidity does not care about what you name your variables; it only looks at the storage slot position.

Keeping this in mind, we need a way to organize our state variables so that we are not overwriting our storage slots. The first way we can do this is with Diamond Storage.

Diamond Storage takes advantage of how many solidity storage slots there are in a smart contract (2^256). The theory behind Diamond Storage is that because there are so many storage slots that if we hash a unique value, we will get a random storage slot that will almost certainly not collide with another storage slot. This may sound risky, but it is actually the same process solidity uses to store mappings and dynamic arrays. Diamond Storage provides the opportunity for your facets to keep state variables specific to their contract, while also allowing facets to share state variables if desired.

Since the Diamond Standard’s complexity requires a bit of setup, for these storage examples I will not be using the Diamond Standard. The main purpose of this section is to understand the concepts of how your state variables are stored. Implementation in the Diamond Standard is mostly the same.
```
pragma solidity^0.8.17;

library SharedLib {

    // struct that with state variable
    struct DiamondStorage {
        uint256 sharedVar;
    }

    // returns a storage variable with our state variable
    function diamondStorage() internal pure returns(DiamondStorage storage ds) {

        // gets a "random" storage positon by hashing a string
        bytes32 storagePosition = keccak256(abi.encode("Diamond.Storage.SharedLib"));

        // assigns our struct storage slot to storage position
        assembly {
            ds.slot := storagePosition
        }
        
    }

}

// not the actual Diamond Standard
contract PseudoDiamond {

    // delegate calls Facet1 to update sharedVar
    function writeToSharedVar(address _facet1, uint256 _value) external {
        // writes via delegate call
        _facet1.delegatecall(
            abi.encodeWithSignature("writeShared(uint256)", _value)
        );
    }

    // delegate calls Facet2 to read sharedVar
    function readSharedVar(address _facet2) external returns (uint256) {
        // returns result of delegate call
        (bool success, bytes memory _valueBytes) = _facet2.delegatecall(
            abi.encodeWithSignature("readShared()")
        );

        // since return value is bytes array we use assembly to retrieve our uint
        bytes32 _value;
        assembly {
            let location := _valueBytes
            _value := mload( add(location, 0x20) )
        }

        return uint256(_value);
    }

}

contract Facet1 {

    function writeShared(uint256 _value) external {

        // initializes the storage struct by calling library function
        SharedLib.DiamondStorage storage ds = SharedLib.diamondStorage();

        // writes to shared variable
        ds.sharedVar = _value;
       
    }
}

contract Facet2 {
   
    function readShared() external view returns (uint256) {

        // initializes the storage struct by calling library function
        SharedLib.DiamondStorage storage ds = SharedLib.diamondStorage();


        // returns shared variable
        return ds.sharedVar;
        
    }
}
```
If you call ```PseudoDiamond.writeSharedVar()```, then ```PseudoDiamond.readSharedVar()``` you will see your value. By using a library and indexing a “random” storage slot we can share variables between the two smart contracts. When we ```delegatecall()``` both facets, it is looking at the storage position of the ```DiamondStorage``` struct to access that variable. Through explicit communication between facets of where we want to store our data, we prevent state variable collisions. If you want to have state variables for only one of your facets you can simply create a library, similar to ```SharedLib```, and only implement it into that specific facet. 


App Storage works a little bit differently. You create a Soldity file, and inside that file you create a struct called ```AppStorage```. You then place as many state variables as you wish inside that struct, including other structs. Then the first thing you do inside your smart contracts is initialize ```AppStorage```. This is setting storage slot 0 to the beginning of the struct. This creates a human readable shared state between contracts. Let’s look at an example!
```
pragma solidity^0.8.17;

// struct with state variable
struct StateVars {
    uint256 sharedVar;
}

// our app storage struct
struct AppStorage {
    StateVars state;
}

// not the actual Diamond Standard
contract PseudoDiamond {

    AppStorage s;

    // delegate calls Facet1 to update sharedVar
    function writeToSharedVar(address _facet1, uint256 _value) external {
        // writes via delegate call
        _facet1.delegatecall(
            abi.encodeWithSignature("writeShared(uint256)", _value)
        );
    }

    // delegate calls Facet2 to read sharedVar
    function readSharedVar(address _facet2) external returns (uint256) {
        // returns result of delegate call
        (bool success, bytes memory _valueBytes) = _facet2.delegatecall(
            abi.encodeWithSignature("readShared()")
        );

        // since return value is bytes array we use assembly to retrieve our uint
        bytes32 _value;
        assembly {
            let location := _valueBytes
            _value := mload( add(location, 0x20) )
        }

        return uint256(_value);
    }

}

contract Facet1 {

    AppStorage s;

    function writeShared(uint256 _value) external {
        // writes to our state variable in storage
        s.state.sharedVar = _value;
    }
}

contract Facet2 {

    AppStorage s;
   
    function readShared() external view returns (uint256) {
        // returns state variable from app storage
        return s.state.sharedVar;
    }
}
```
If you look at the value in storage slot 0 of ```PseudoDiamond```, you see 100 which is our value! An important note on App Storage is if you need to update your AppStorage after deployment make sure to add your new state variables at the end of AppStorage to prevent storage collisions. Personally, I prefer App Storage to Diamond Storage for the organization, but both get the job done. 
