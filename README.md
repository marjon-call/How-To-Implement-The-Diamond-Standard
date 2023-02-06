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



## App Storage & Diamond Storage
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
If you look at the value in storage slot 0 of ```PseudoDiamond```, you see 100 which is our value! An important note on App Storage is if you need to update your AppStorage after deployment make sure to add your new state variables at the end of AppStorage to prevent storage collisions. Personally, I prefer App Storage to Diamond Storage for the organization, but both get the job done. It is also worth noting that Diamond Storage and App Storage are not exclusive. Even if you use App Storage to manage your state variables ```LibDiamond.sol```, uses Diamond Storage to manage facet data.

## Diamond.sol
As previously mentioned, ```Diamond.sol``` is the smart contract that gets called when interacting with your application. If it helps to think of it this way, ```Diamond.sol``` “manages” the rest of your application. All facet functions will appear to be ```Diamond.sol```’s own functions.

Let’s first take a look at what is happening in the constructor of ```diamond-1```.
```
// This is used in diamond constructor
// more arguments are added to this struct
// this avoids stack too deep errors
struct DiamondArgs {
    address owner;
    address init;
    bytes initCalldata;
}

contract Diamond {    

    constructor(IDiamondCut.FacetCut[] memory _diamondCut, DiamondArgs memory _args) payable {
        LibDiamond.setContractOwner(_args.owner);
        LibDiamond.diamondCut(_diamondCut, _args.init, _args.initCalldata);

        // Code can be added here to perform actions and set state variables.
    }

    // rest of code
}
```
First, outside of the contract, we have a struct that formats our data. Next, we are passing that struct as a parameter along with another struct that looks like this:
```
struct FacetCut {
    address facetAddress;
    FacetCutAction action;
    bytes4[] functionSelectors;
}
```
```FacetCut``` is providing the address of the facet, the action we want to perform on the facet (add, replace, or remove), and the function’s selectors for the facet’s functions. Next, we set the contract owner, and add whatever facet data was provided to the Diamond. We will go over in more detail how this works later on. After that, if we wanted to, we can initialize state variables. Keep in mind you should never perform any state variable assignments in the constructor of a facet, because it would perform that operation inside the facet smart contract and not ```Diamond.sol```.

Seems pretty straightforward so far so let’s take a look at what’s happening in the constructor for ```diamond-2``` and ```diamond-3```.
```
contract Diamond {    

    constructor(address _contractOwner, address _diamondCutFacet) payable {        
        LibDiamond.setContractOwner(_contractOwner);

        // Add the diamondCut external function from the diamondCutFacet
        IDiamondCut.FacetCut[] memory cut = new IDiamondCut.FacetCut[](1);
        bytes4[] memory functionSelectors = new bytes4[](1);
        functionSelectors[0] = IDiamondCut.diamondCut.selector;
        cut[0] = IDiamondCut.FacetCut({
            facetAddress: _diamondCutFacet,
            action: IDiamondCut.FacetCutAction.Add,
            functionSelectors: functionSelectors
        });

        LibDiamond.diamondCut(cut, address(0), "");        
    }

    // rest of code

}
```
This time we only pass in the owner of the contract, and the address of ```DiamondCutFacet.sol```. Next, we are assigning the owner of the Diamond. After that we are adding the ```diamondCut()``` function so that we can add more facets. We do this by first initializing a memory array with one element of the struct type ```FacetCut```, which we saw earlier. Then we are initializing another array with one element. This time the array is ```bytes4```, and is used to store ```diamondCut()```’s function selector. Next we assign the function selector to the array by calling the Diamond Cut interface’s function ```diamondCut```, and getting its selector. After that we can assign ```cut[0]```. We use the address of ```DiamondCutFacet.sol```, add (because we want to add this facet to our Diamond), and the function selector array. Finally, we actually add ```diamondCut```. This is a lot not knowing what happens under the hood, so if it helps you can reread this section after we go over ```DiamondCut.sol```. For now, just understand that we are adding facets in the constructor.

The last part of ```Diamond.sol``` is the ```fallback()``` function. This is how we will be calling our facets with the Diamond Standard. It looks almost the same in all three implementations of the Diamond Standard, so let’s go over it!

```
// Find facet for function that is called and execute the
// function if a facet is found and return any value.
fallback() external payable {
    LibDiamond.DiamondStorage storage ds;
    bytes32 position = LibDiamond.DIAMOND_STORAGE_POSITION;
    // get diamond storage
    assembly {
        ds.slot := position
    }
    // get facet from function selector
    address facet = address(bytes20(ds.facets[msg.sig]));
    require(facet != address(0), "Diamond: Function does not exist");
    // Execute external function from facet using delegatecall and return any value.
    assembly {
        // copy function selector and any arguments
        calldatacopy(0, 0, calldatasize())
        // execute function call using the facet
        let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
        // get any return value
        returndatacopy(0, 0, returndatasize())
        // return any return value or error back to the caller
        switch result
            case 0 {
                revert(0, returndatasize())
            }
            default {
                return(0, returndatasize())
            }
    }
}
```
The first thing we are doing is initializing Diamond Storage. This is where we will store our facet’s data. Next is where the difference in the ```fallback()```s resides. We are looking at ```diamond-2``` above. In all three we are checking if the address of the facet exists. <br>
Here is how we check in ```diamond-1```:
```
// get facet from function selector
address facet = ds.facetAddressAndSelectorPosition[msg.sig].facetAddress;
if(facet == address(0)) {
    revert FunctionNotFound(msg.sig);
}
```
Here is how we check in  ```diamond-3```:
```
// get facet from function selector
address facet = ds.selectorToFacetAndPosition[msg.sig].facetAddress;
require(facet != address(0), "Diamond: Function does not exist");
```
The main difference in the three implementations is that in ```diamond-2``` we store the selectors in a mapping of 32 byte storage slots.

Finally, we use Yul to ```delegatecall()``` our facets by copying the calldata that was sent to ```Diamond.sol```, and checking if the call was successful.

This wraps up our section on ```Diamond.sol```. Next, we will look into what was happening when we called ```diamondCut()```.


## DiamondCut.sol
As we have already discussed, ```DiamondCut.sol``` is responsible for adding, removing, and replacing facets in our Diamond. All three implementations take a slightly different approach, but accomplish the same goal. Let’s look into how ```diamond-1``` works. 
```
contract DiamondCutFacet is IDiamondCut {
    /// @notice Add/replace/remove any number of functions and optionally execute
    ///         a function with delegatecall
    /// @param _diamondCut Contains the facet addresses and function selectors
    /// @param _init The address of the contract or facet to execute _calldata
    /// @param _calldata A function call, including function selector and arguments
    ///                  _calldata is executed with delegatecall on _init
    function diamondCut(
        FacetCut[] calldata _diamondCut,
        address _init,
        bytes calldata _calldata
    ) external override {
        LibDiamond.enforceIsContractOwner();
        LibDiamond.diamondCut(_diamondCut, _init, _calldata);
    }
}
```
As you can see, this contract heavily relies on ```LibDiamond.sol```. All we are doing, from a high level overview, is checking that the contract owner has made this call, and then call ```diamondCut()```. Let’s look at what’s happening in ```LibDiamond.sol```. Before we do, I need to point out one big detail about ```LibDiamond.sol```. ```LibDiamond.sol``` solely uses ```internal``` functions. This adds the bytecode to our contract, saving us from needing to use another ```delegatecall()```. Ok, now that we understand how we save some gas costs in ```LibDiamond.sol```, let's look at the code of ```diamondCut()```.
```
// Internal function version of diamondCut
function diamondCut(
    IDiamondCut.FacetCut[] memory _diamondCut,
    address _init,
    bytes memory _calldata
) internal {
    for (uint256 facetIndex; facetIndex < _diamondCut.length; facetIndex++) {
        bytes4[] memory functionSelectors = _diamondCut[facetIndex].functionSelectors;
        address facetAddress = _diamondCut[facetIndex].facetAddress;
        if(functionSelectors.length == 0) {
            revert NoSelectorsProvidedForFacetForCut(facetAddress);
        }
        IDiamondCut.FacetCutAction action = _diamondCut[facetIndex].action;
        if (action == IDiamond.FacetCutAction.Add) {
            addFunctions(facetAddress, functionSelectors);
        } else if (action == IDiamond.FacetCutAction.Replace) {
            replaceFunctions(facetAddress, functionSelectors);
        } else if (action == IDiamond.FacetCutAction.Remove) {
            removeFunctions(facetAddress, functionSelectors);
        } else {
            revert IncorrectFacetCutAction(uint8(action));
        }
    }
    emit DiamondCut(_diamondCut, _init, _calldata);
    initializeDiamondCut(_init, _calldata);
}


```
We take the same parameters as before for this function. First, we loop through our ```FacetCut``` struct. Inside the loop, we are getting our function selector and address of the facet function. We make sure the facet selector is valid, and revert otherwise. Next, we need to check the action we are performing on this specific function (add, replace, or remove). After finding the action, we call the helper function that correlates with that action. Then we emit an event to provide transparency to our users. Finally, we verify that our facet works with ```initializeDiamondCut()``` by checking if our contract has code, and can be called via```delegatecall()``` with ```_calldata```.

Now that we know how ```diamondCut()``` works, let’s see how we perform each action starting with ```Add```.
```
function addFunctions(address _facetAddress, bytes4[] memory _functionSelectors) internal {        
    if(_facetAddress == address(0)) {
        revert CannotAddSelectorsToZeroAddress(_functionSelectors);
    }
    DiamondStorage storage ds = diamondStorage();
    uint16 selectorCount = uint16(ds.selectors.length);                
    enforceHasContractCode(_facetAddress, "LibDiamondCut: Add facet has no code");
    for (uint256 selectorIndex; selectorIndex < _functionSelectors.length; selectorIndex++) {
        bytes4 selector = _functionSelectors[selectorIndex];
        address oldFacetAddress = ds.facetAddressAndSelectorPosition[selector].facetAddress;
        if(oldFacetAddress != address(0)) {
            revert CannotAddFunctionToDiamondThatAlreadyExists(selector);
        }            
        ds.facetAddressAndSelectorPosition[selector] = FacetAddressAndSelectorPosition(_facetAddress, selectorCount);
        ds.selectors.push(selector);
        selectorCount++;
    }
}
```
The input parameters are the facet contract’s address and the particular function we are working withs selector. We want to verify this contract exists by checking if the address is the zero address. Next, we initialize Diamond Storage. Then, we get the amount of selectors our Diamond already has. After, we check if our facet has contract code. Now, we need to verify that this function is not already in the Diamond. We do this by looping through the function selectors and checking if the address exists already, Otherwise, we are pushing our selector to our stored selectors.

Now, let’s look at how to replace a facet!
```
function replaceFunctions(address _facetAddress, bytes4[] memory _functionSelectors) internal {        
    DiamondStorage storage ds = diamondStorage();
    if(_facetAddress == address(0)) {
        revert CannotReplaceFunctionsFromFacetWithZeroAddress(_functionSelectors);
    }
    enforceHasContractCode(_facetAddress, "LibDiamondCut: Replace facet has no code");
    for (uint256 selectorIndex; selectorIndex < _functionSelectors.length; selectorIndex++) {
        bytes4 selector = _functionSelectors[selectorIndex];
        address oldFacetAddress = ds.facetAddressAndSelectorPosition[selector].facetAddress;
        // can't replace immutable functions -- functions defined directly in the diamond in this case
        if(oldFacetAddress == address(this)) {
            revert CannotReplaceImmutableFunction(selector);
        }
        if(oldFacetAddress == _facetAddress) {
            revert CannotReplaceFunctionWithTheSameFunctionFromTheSameFacet(selector);
        }
        if(oldFacetAddress == address(0)) {
            revert CannotReplaceFunctionThatDoesNotExists(selector);
        }
        // replace old facet address
        ds.facetAddressAndSelectorPosition[selector].facetAddress = _facetAddress;
    }
}


```
As you may have noticed ```Replace``` starts off the same way as ```Add```. We initialize Diamond Storage, check if the facet has a valid address and code size, then loop through the selectors. First, we check if the selector is immutable. Then, we check if the function we want to replace is the same function we are adding. After, we check if the facet address is valid. Otherwise, we replace our facet.

Now, let’s check out our last action, ```Remove```.
```
function removeFunctions(address _facetAddress, bytes4[] memory _functionSelectors) internal {        
    DiamondStorage storage ds = diamondStorage();
    uint256 selectorCount = ds.selectors.length;
    if(_facetAddress != address(0)) {
        revert RemoveFacetAddressMustBeZeroAddress(_facetAddress);
    }        
    for (uint256 selectorIndex; selectorIndex < _functionSelectors.length; selectorIndex++) {
        bytes4 selector = _functionSelectors[selectorIndex];
        FacetAddressAndSelectorPosition memory oldFacetAddressAndSelectorPosition = ds.facetAddressAndSelectorPosition[selector];
        if(oldFacetAddressAndSelectorPosition.facetAddress == address(0)) {
            revert CannotRemoveFunctionThatDoesNotExist(selector);
        }
       
       
        // can't remove immutable functions -- functions defined directly in the diamond
        if(oldFacetAddressAndSelectorPosition.facetAddress == address(this)) {
            revert CannotRemoveImmutableFunction(selector);
        }
        // replace selector with last selector
        selectorCount--;
        if (oldFacetAddressAndSelectorPosition.selectorPosition != selectorCount) {
            bytes4 lastSelector = ds.selectors[selectorCount];
            ds.selectors[oldFacetAddressAndSelectorPosition.selectorPosition] = lastSelector;
            ds.facetAddressAndSelectorPosition[lastSelector].selectorPosition = oldFacetAddressAndSelectorPosition.selectorPosition;
        }
        // delete last selector
        ds.selectors.pop();
        delete ds.facetAddressAndSelectorPosition[selector];
    }
}
``` 
Again, we start by initializing Diamond Storage, checking if the facet has a valid address, then loop through the selectors. Next we get the function selector and position in storage. We verify that it actually exists, then, verify it isn’t immutable. After, we move our selector to the end of the array and perform a ```pop()``` to remove it.

```diamond-2``` and ```diamond-3``` both obtain the same goal as ```diamond-1```’s ```diamondCut()``` function, but use different syntax and architecture. For the simplicity of this article we will not be going over them. However, if enough people are interested in learning about how they work I can make a new article in the future that will go into more detail on the differences in the implementations.



## LoupeFacet.sol
Now that we understand how to update functions in our Diamond, let’s go over how we can view our facets. Remember, for ```diamond-1``` and ```diamond-2``` it is recommended to not call these functions on chain. ```diamond-3```, however, is heavily optimized for calling these functions on chain. Again, we will only be going over ```diamond-1```, but if you understand what is happening, you should be able to both comprehend and implement any of the other implementations of the Diamond Standard.

We will first look at ```facets()```. ```facets()```, returns all of the facets and their selectors for a Diamond.
```
function facets() external override view returns (Facet[] memory facets_) {
    LibDiamond.DiamondStorage storage ds = LibDiamond.diamondStorage();
    uint256 selectorCount = ds.selectors.length;
    // create an array set to the maximum size possible
    facets_ = new Facet[](selectorCount);
    // create an array for counting the number of selectors for each facet
    uint16[] memory numFacetSelectors = new uint16[](selectorCount);
    // total number of facets
    uint256 numFacets;
    // loop through function selectors
    for (uint256 selectorIndex; selectorIndex < selectorCount; selectorIndex++) {
        bytes4 selector = ds.selectors[selectorIndex];
        address facetAddress_ = ds.facetAddressAndSelectorPosition[selector].facetAddress;
        bool continueLoop = false;
        // find the functionSelectors array for selector and add selector to it
        for (uint256 facetIndex; facetIndex < numFacets; facetIndex++) {
            if (facets_[facetIndex].facetAddress == facetAddress_) {
                facets_[facetIndex].functionSelectors[numFacetSelectors[facetIndex]] = selector;                                  
                numFacetSelectors[facetIndex]++;
                continueLoop = true;
                break;
            }
        }
        // if functionSelectors array exists for selector then continue loop
        if (continueLoop) {
            continueLoop = false;
            continue;
        }
        // create a new functionSelectors array for selector
        facets_[numFacets].facetAddress = facetAddress_;
        facets_[numFacets].functionSelectors = new bytes4[](selectorCount);
        facets_[numFacets].functionSelectors[0] = selector;
        numFacetSelectors[numFacets] = 1;
        numFacets++;
    }
    for (uint256 facetIndex; facetIndex < numFacets; facetIndex++) {
        uint256 numSelectors = numFacetSelectors[facetIndex];
        bytes4[] memory selectors = facets_[facetIndex].functionSelectors;
        // setting the number of selectors
        assembly {
            mstore(selectors, numSelectors)
        }
    }
    // setting the number of facets
    assembly {
        mstore(facets_, numFacets)
    }
}
```
Notice there are no input parameters, and we specify that we will be returning ```factes_```. ```factes_``` is a data structure that looks like this:
```
struct Facet {
    address facetAddress;
    bytes4[] functionSelectors;
}
```
The first thing we do inside the function is initialize Diamond Storage. Next, we retrieve the number of selectors that we have. After, we initialize the array we will be returning at the end of our function. Then, we create an array to keep track of how many functions per facet, and a variable to keep track of the number of facets. Next, we loop through our function selectors. Inside our loop, we are looking to find what facet our selector belongs too. We have to loop through the facets to find which address matches our current function selectors’ address. After we find our facet, we add our function selector to that facet’s array if it exists. Otherwise, we create the array. After we finish our loop, we loop through the facets one more time. Inside this loop, we are storing the number of selectors to memory to return later on. Finally, we store the number of facets and return. The reason we are storing the number of selectors and facets is because we originally initialized our arrays to the maximum size possible. Now that we know how many specific selectors and facets we will be returning, we tell solidity the correct array size to return.

The next function we will look at is ```facetFunctionSelectors()```. ```facetFunctionSelectors()``` returns the function selectors for a particular facet. It takes as a parameter the address of the targeted facet, and returns a ```bytes4[]``` array representing the function selectors.
```
function facetFunctionSelectors(address _facet) external override view returns (bytes4[] memory _facetFunctionSelectors) {
    LibDiamond.DiamondStorage storage ds = LibDiamond.diamondStorage();
    uint256 selectorCount = ds.selectors.length;
    uint256 numSelectors;
    _facetFunctionSelectors = new bytes4[](selectorCount);
    // loop through function selectors
    for (uint256 selectorIndex; selectorIndex < selectorCount; selectorIndex++) {
        bytes4 selector = ds.selectors[selectorIndex];
        address facetAddress_ = ds.facetAddressAndSelectorPosition[selector].facetAddress;
        if (_facet == facetAddress_) {
            _facetFunctionSelectors[numSelectors] = selector;
            numSelectors++;
        }
    }
    // Set the number of selectors in the array
    assembly {
        mstore(_facetFunctionSelectors, numSelectors)
    }
}
```
Again we initialize Diamond Storage. Then, we get the number of function selectors, and initialize our return array. Next, we loop through the selectors. Here, we are checking if the selector address is the same as our targeted facet. If it is, we store that function selector. Finally, we store the number of selectors and return.

 Now we will look over ```facetAddresses()```, which returns an array of addresses for our Diamond’s facets. 
```
function facetAddresses() external override view returns (address[] memory facetAddresses_) {
    LibDiamond.DiamondStorage storage ds = LibDiamond.diamondStorage();
    uint256 selectorCount = ds.selectors.length;
    // create an array set to the maximum size possible
    facetAddresses_ = new address[](selectorCount);
    uint256 numFacets;
    // loop through function selectors
    for (uint256 selectorIndex; selectorIndex < selectorCount; selectorIndex++) {
        bytes4 selector = ds.selectors[selectorIndex];
        address facetAddress_ = ds.facetAddressAndSelectorPosition[selector].facetAddress;
        bool continueLoop = false;
        // see if we have collected the address already and break out of loop if we have
        for (uint256 facetIndex; facetIndex < numFacets; facetIndex++) {
            if (facetAddress_ == facetAddresses_[facetIndex]) {
                continueLoop = true;
                break;
            }
        }
        // continue loop if we already have the address
        if (continueLoop) {
            continueLoop = false;
            continue;
        }
        // include address
        facetAddresses_[numFacets] = facetAddress_;
        numFacets++;
    }
    // Set the number of facet addresses in the array
    assembly {
        mstore(facetAddresses_, numFacets)
    }
}
```
Again, the first thing we do is initialize Diamond Storage, get the number of function selectors, and initialize our return array. We are also looping through our selectors again. We are checking what the facet address of our selector is. Then, we check if we have seen this address before. If we have, we skip this iteration of the loop. Otherwise, we are adding this new address to our return array. Again, we update the size of the array and return it.

The last function we will look at from ```DiamondLoupeFacet.sol``` is ```facetAddress```. This function returns the address of a facet provided a function selector.
```
function facetAddress(bytes4 _functionSelector) external override view returns (address facetAddress_) {
    LibDiamond.DiamondStorage storage ds = LibDiamond.diamondStorage();
    facetAddress_ = ds.facetAddressAndSelectorPosition[_functionSelector].facetAddress;
}
```
This function is pretty simple. We initialize Diamond Storage and take advantage of how we store our selectors, to return our facet address.

This wraps up our section on the loupe facet. We now know how the Diamond Standard provides transparency for its users. Next we will look at how deploying the Diamond Standard works.

## How To Deploy The Diamond Standard
When deploying a Diamond, you need to deploy the facets and then deploy ```Diamond.sol```. This way you can tell your Diamond what contracts it will be calling. The easiest way to deploy a Diamond is with the npm package ```diamond-util```. Here is how to install it: <br>
```
npm i diamond-util
```
Once you have it installed, you can deploy your Diamond with the following code.
```
// eslint-disable-next-line no-unused-vars
const deployedDiamond = await diamond.deploy({
  diamondName: 'LeaseDiamond',
  facets: [
    'DiamondCutFacet',
    'DiamondLoupeFacet',
    'Facet1',
    'Facet2'
  ],
  args: [/* your parameters */]
})


// init facets
const diamondCutFacet = await ethers.getContractAt('DiamondCutFacet', deployedDiamond.address)
const diamondLoupeFacet = await ethers.getContractAt('DiamondLoupeFacet', deployedDiamond.address)
const facet1 = await ethers.getContractAt('Facet1', deployedDiamond.address)
const facet2 = await ethers.getContractAt('Facet2', deployedDiamond.address)
```
This library takes care of most of the work for us! All we have to do is list our facets and the library deploys them along with the Diamond. Take note that when we initialize our contracts with ```ethers.js``` we are setting the address to ```Diamond.sol```’s address.

If you want to add a new facet, you can do so by calling ```DiamondCutFacet.sol```. Here is an example.
```
const FacetCutAction = { Add: 0, Replace: 1, Remove: 2 }


const Facet3 = await ethers.getContractFactory('Facet3')
const facet3 = await Facet3.deploy()
await facet3.deployed()
const selectors = getSelectors(facet3).remove(['supportsInterface(bytes4)'])
tx = await diamondCutFacet.diamondCut(
  [{
    facetAddress: facet3.address,
    action: FacetCutAction.Add,
    functionSelectors: selectors
  }],
  ethers.constants.AddressZero, '0x', { gasLimit: 800000 }
)


receipt = await tx.wait()
```
As you see we deploy our facet first. Then we use our npm package to get the selectors of the functions. Then we call ```DiamondCutFacet.sol``` to update our Diamond. ```Replace``` works similarly, except you have to make sure the selectors you are replacing are already in the Diamond. ```Remove```, also works similarly, but make sure the selectors you pass in are the ones you want to  remove.

Congratulations! You now know how to create and deploy a blockchain application using the Diamond Standard!


## Conclusion
This concludes my article on the Diamond Standard. Hopefully I have helped you to understand the complexities of the Diamond Standard, and how to implement it in your own projects.

For further reading on the Diamond Standard check out these links. <br>
EIP-2535: https://eips.ethereum.org/EIPS/eip-2535 <br>
To read up on different Diamond Standard implementations: https://github.com/mudgen/diamond <br>
```diamond-1``` template: https://github.com/mudgen/diamond-1-hardhat <br>
```diamond-2``` template: https://github.com/mudgen/diamond-2-hardhat <br>
```diamond-3``` template: https://github.com/mudgen/diamond-3-hardhat <br>
Gas Benefits of Diamond Standard: https://eip2535diamonds.substack.com/p/how-eip2535-diamonds-reduces-gas?s=w <br>
To read Nick Mudge’s blog posts on the Diamond Standard: https://eip2535diamonds.substack.com/ <br>
To listen to Nick Miudge walk through the Diamond Standard in video form: https://www.youtube.com/watch?v=9-MYz75FA8o <br>
If you write, or have written, a smart contract with the Diamond Standard and want it audited by Nick Mudge’s auditing firm, check out their website: https://www.perfectabstractions.com/ <br>


If you have any questions, or would like to see me make a tutorial on a different topic please leave a comment below. <br>

If you would like to support me making tutorials, here is my Ethereum address: 0xD5FC495fC6C0FF327c1E4e3Bccc4B5987e256794.

