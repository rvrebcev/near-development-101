# NEAR development 101 - Smart contract development

## Setup development environment & project bootstrap

Before jumping to the smart contract itself let's set up a development environment.

### CLI tools

As long as we are going to use a terminal heavily some CLI tools must be installed:
* `node` - NodeJS engine that is needed to run applications, in this case - a ReactJS web application. NodeJS 12 or higher is required.
* `yarn` - is a package manager that we are going to use to build smart contracts
* `near-cli` - a CLI tool for NEAR that offers an API to interact with smart contracts
* `assemblyscript` - a smart contracts language 
* `asbuild` - a build tool for AssemblyScript

There are no preferences regarding tools versions so we can just install the latest available GA releases.

For more convenience we install some tools globally:
```bash=
yarn add global near-cli
yarn add global assemblyscript
yarn add global asbuild
```

For a code editor, you can use any IDE/code editor that supports syntax highlighting for TypeScript so it will work with AssemblyScript too.

The next step is to prepare a project structure.

### Create project structure
The next directories and files must be created to proceed with smart contracts:
* assembly/ - this directory contains smart contracts source code
* asconfig.json - contains most of the configuration properties
* assembly/tsconfig.json

#### `asconfig.json`
This config provides CLI options and configurations for AssemblyScript like compile targets and whatnot.
By default, it's needed to add the next content to the file. By adding this, we just extend the configuration provided by `near-sdk-as`.
```javascript=
{
    "extends": "near-sdk-as/asconfig.json"
}
```

#### `assembly/tsconfig.json`
The purpose of this file is to specify compiler options and root level files that are necessary for a TypeScript project to be compiled.
Also, this file implies that the directory where `tsconfig.json` is located in the root of the TypeScript project.
```javascript=
{
  "extends": "../node_modules/assemblyscript/std/assembly.json",
  "include": [
    "./**/*.ts"
  ]
}
```

#### `assembly/as_types.d.ts`
This files declares that some type names must be included in the compilation. In this case, names are imported from `near-sdk-as`
```javascript=
/// <reference types="near-sdk-as/assembly/as_types" />
```

### Initialize project

Run the next commands in a terminal window (in the project's root):
```bash=
yarn init
```
It will create a `package.json` file where development dependencies can be added.


Run the next command to add `near-sdk-as` to the project:
```bash=
yarn add -D near-sdk-as
```

The next step is to create an entry file for the smart contract - create the `index.ts` file in the `assembly` directory.
The resulting project structure should be like this:
```
├── asconfig.json
├── assembly
│   ├── as_types.d.ts
│   ├── index.ts
│   └── tsconfig.json
├── package.json
└── yarn.lock
```

Now we are set up to develop our first smart contract in AssemblyScript and deploy it in NEAR.

## Smart contract implementation

As we have chosen AssemblyScript for the sake of this course, writing a smart contract using this language is very similar to developing apps using TypeScript.

Although, there are a couple of things to remember when developing a smart contract for NEAR.

### Storage
When it comes to building purposeful smart contracts you might need some storage to keep data. NEAR offers multiple options on how to store data and all of them can be a good fit for different use cases.

There is a `Storage` class that represents a datastore on the blockchain. It offers a pretty simple API to interact with.
Also, multiple collections offer a high-level API to store and retrieve data:
* `PersistentMap` - a key-value data structure common for most languages
* `PersistentVector` - an array-like data structure
* `PersistentDeque` - a bidirectional queue
* `PersistentUnorderedMap` - similar to `PersistentMap` but with useful functions that allow us to iterate over keys, values, entries.

In this course, we are going to use `PersistentMap` and `PersistentUnorderedMap` for different purposes.

More details about storage and collections can be found [here](https://docs.near.org/docs/develop/contracts/as/intro#state-and-data).

### Read & write state
There are two types of function calls - `view` and `change`. The former one just reads state and no modifications are done hence it's free. For instance, if you call `PersistentUnorderedMap#getSome(key: K)`, it will cost nothing to retrieve data behind the corresponding key.

The latter one modifies state and gas must be paid for the transaction to be completed - when `PersistentUnorderedMap#set(key: K, value: V)` is called, it impacts the storage.

Next, we implement our first function that can store something on the blockchain. We implement these functions in the `assembly/index.ts` file.

#### `change` function
Let's begin with the mandatory imports. First of all, we need to import `PersistentUnorderedMap` from the `near-sdk-as`:
```javascript=
import { PersistentUnorderedMap } from "near-sdk-as";
```
Once it's imported, we can use that data structure to store data on the blockchain.

Next, we need to define a function that would be called to store data. 

Every function that is supposed to be invoked from the outside must be exported.
So we define a simple write function like this:
```javascript=
export function setProduct(productName: string, amount: int): void {
    
}
```
where `productName` is the parameter that states for a product name to be stored and `id` is an id the of the product.

Also, the return type is `void` because we are returning nothing after changing the state.

The next step is to initialize a map and put `productName` to the storage:
```javascript=
export const products = new PersistentUnorderedMap<string, string>("PRODUCTS");
products.set(id, productName)
```

The string `LISTED_PRODUCTS` in the `PersistentUnorderedMap`'s constructor is the unique prefix to use for every key.

The complete snippet should look like this:
```javascript=
import { PersistentUnorderedMap } from "near-sdk-as";

export const products = new PersistentUnorderedMap<string, string>("PRODUCTS");

export function setProduct(id: string, productName: string): void {
    products.set(id, productName);
}
```

#### `view` function

Now let's read data from the blockchain. The definition of the read function is pretty simple:
```javascript=
export function getProduct(id: string): string | null {
    return products.get(id);
}
```
In this case, when we read state, the return type is not `void` anymore (as long as we want to return something from that function).
Here we have `string | null` as the return type and it means that the type of the value that we return from the function is `string`. 

The second part `| null` means that if there is no data behind the given key in the `PersistentUnorderedMap`, we just return `null` and that's it.

## Compile, build, and deploy the smart contract

Let's build, deploy and test `setProduct` and `getProduct` methods before we added more functionalities.

First of all, we need to create an account and log in to the shell with it.

### Create account
To test the contract we would need two accounts.

The first account is the account that you can use to interact with a contract.

The second account would be a subaccount of the first one. In practice, it looks like a subdomain.

For example, you create an account `myaccount.testnet` and a subaccount for a smart contract would look like this: `mycontract.myaccount.testnet`.

Let's navigate to the NEAR `testnet` wallet [NEAR Testnet wallet](https://wallet.testnet.near.org/) and create a top-level account:
* Open the wallet
* Choose a name for your account
* Choose a security method. For this course, we will use a passphrase method
* Copy and save somewhere safe a passphrase
* At the next step, you will be asked to enter of the words from the passphrase

That's it, your test account is ready to use. There is no need for such a thing as a faucet - all test accounts are created with test tokens allocated.

Currently, it's not possible to create a subaccount using a wallet so we can do this in the shell using `near-cli`.

![](https://github.com/dacadeorg/near-development-101/raw/master/content/gifs/create_account.gif)

### Login to an account in a shell
To deploy a contract via terminal, the account's credentials are needed. 
To get the credentials, run the next command in a terminal window:
```bash=
near login
```
It opens a wallet URL in a browser where you can log in to your account (or select one of the existing accounts if you have one). 

Next, you will be asked to grant permissions to the shell. We will need it to be able to deploy a smart contract and to call functions.

As the result, the session in the terminal window is authenticated and you can start deploying contracts and view/call functions.

![](https://github.com/dacadeorg/near-development-101/raw/master/content/gifs/login_to_shell.gif)


### Create a subaccount for a smart contract
Now that we have a top-level account we can create a subaccount and `near-cli` offers an API for this:
```bash=
near create-account ${CONTRACT_NAME}.${ACCOUNT_ID} --masterAccount ${ACCOUNT_ID} --initialBalance ${INITIAL_BALANCE}
```
where:
* `CONTRACT_NAME` - any contract name of your preferrence
* `ACCOUNT_ID` - id of a top-level account
* `INITIAL_BALANCE` - initial balance for a subaccount

The actual invocation should look like this:
```bash=
near create-account mycontract.myaccount.testnet --masterAccount myaccount.testnet --initialBalance 50
```

Now that we have a subaccount let's jump to building and deployment.

### Compile & build a smart contract
Before a smart contract can be deployed, it must be built as `.wasm`. 
To do that, the next command should be run from the project's root:
```bash=
yarn asb
```

The output of this command is a `.wasm` file that is placed into `${PROJECT_ROOT}/build/release/{CONTRACT_NAME}.wasm` directory where `{CONTRACT_NAME}` is the same as the directory name where the smart contract is located.

### Deploy a smart contract
To deploy a smart contract, run the next command from in a terminal window:
```bash=
near deploy --accountId=${ACCOUNT_ID} --wasmFile=${PATH_TO_WASM}
```
where:
* `${ACCOUNT_ID}` - an account name that should be used to deploy a smart contract
* `${PATH_TO_WASM}` - a path to the `.wasm` file issued by the `yarn asb` command - `${PROJECT_ROOT}/build/release/${WASM_FILE_NAME}.wasm` `${WASM_FILE_NAME}` has the same name as the root directory of a smart contract.

### Contract interaction in the terminal

So the contract is deployed we can invoke `call` and `view` methods that change and read data respectively.

#### `near call` 

First, we are going to invoke a `call` function.

The interface to that call looks like this:
```bash=
near call ${CONTRACT_NAME} ${METHOD_NAME} ${PAYLOAD} --accountId=${ACCOUNT_ID}
```

Now let's replace placeholder values.

```bash=
near call mycontract.myaccount.testnet setProduct '{"productName": "tea", "id": "1"}' --accountId=myaccount.testnet
```

If you use PowerSchell or CMD on Windows, you might need to pass the payload using double-quotes only escaping inner ones that wrap properties names and values:
```bash=
near call mycontract.myaccount.testnet writeProduct "{\"productName\": \"tea\", \"id\": \"1\"}" --accountId=myaccount.testnet
```

where `mycontract.myaccount.testnet` is the contract name name from the [Create a subaccount for a smart contract](#Create-a-subaccount-for-a-smart-contract)  step.

`setProduct` is the name of the method that we are calling and the JSON that follows it is a payload that has the same property names as it's defined in the method's signature.

And the last part is `--accountId` which in our case is the same as `CONTRACT_NAME`.

If the call is successful, you should see a response similar to this:
```
Scheduling a call: mycontract.myaccount.testnet.setProduct({"productName": "tea", "id": "1"})
Doing account.functionCall()
Transaction Id 5fVaRBYUjiX3HaYu14Sm1StfUQLMrsr9CvPKht7PQDQi
To see the transaction in the transaction explorer, please open this url in your browser
https://explorer.testnet.near.org/transactions/5fVaRBYUjiX3HaYu14Sm1StfUQLMrsr9CvPKht7PQDQi
''
```

#### `near view` 

Now that we added a product to the blockchain, we can invoke a `view` function.

The interface to that call looks like this:
```bash=
near view ${CONTRACT_NAME} ${METHOD_NAME} ${PAYLOAD}
```

Now let's replace placeholder values.

```bash=
near view mycontract.myaccount.testnet getProduct '{"id": "1"}'
```
where `mycontract.myaccount.testnet` is the contract name name from the [Create Account](#Create-account) step.

`getProduct` is the name of the method that we are calling and the JSON that follows it is a payload that contains an only id that is the key in the map where we store products.

For `view` call we do not need to pass an account id.

If the call is successful, you should see a response similar to this:
```
View call: mycontract.myaccount.testnet.getProduct({"id": "1"})
'tea'
```

Now when we are sure that the contract compiles and builds successfully, and we can call all functions, let's extend them a bit. 

#### Add more properties to the product

In most cases having a product's name and its quantity is not enough. We can add more details about a product by defining one via class.

Now we switch to `model.ts` and create a class that would be a definition of a product:
```javascript=
import { PersistentUnorderedMap, u128, context } from "near-sdk-as";

@nearBindgen
export class Product {
    id: string;
    name: string;
    description: string;
    image: string;
    location: string;
    price: u128;
    owner: string;
    sold: u32;
    public static fromPayload(payload: Product): Product {
        const product = new Product();
        product.id = payload.id;
        product.name = payload.name;
        product.description = payload.description;
        product.image = payload.image;
        product.location = payload.location;
        product.price = payload.price;
        product.owner = context.sender;
        return product;
    }
    public incrementSoldAmount(): void {
        this.sold = this.sold + 1;
    }
}

export const listedProducts = new PersistentUnorderedMap<string, Product>("LISTED_PRODUCTS");
```

This class is pretty straightforward except for one thing: new data types.
Here for `sold` and `price` we use `u32` and `u128` datatypes from the
`near-sdk-as`. These are numeric (unsigned integers) data types specific to NEAR. You can still use Typescript's numeric data types however to avoid data type conversions in the smart contract it might be better to use NEAR types.

`u128` allows storing `price`  in yocto-NEAR, where 1 [yocto](https://www.nanotech-now.com/metric-prefix-table.htm) = 10<sup>24</sup> which is a minimum denomination in NEAR.
Do not forget to import `u128` from `near-sdk-as`.

That's pretty much it except for one thing. `Product` is defined outside the entry point `index.ts` and we must tell NEAR to use this class from `model.ts`. 

That is why we added `@nearBindgen` decorator in front of the `Product` class.

Now we should update `change` and `view` functions to accept and return `Product` accordingly.

Let's just skip methods `fromPayload` and `incrementSoldAmount`. We will come back to them later. 

The `context` object contains some extra information about a transaction. For instance, it holds a sender account and the attached deposit and we are going to use it to get the owner of the transaction - `context.owner`.

The second one is `sold` which is mapped via `incrementSoldAmount()` function whenever a product is sold.

Also, we create `listedProducts` map (in `model.ts`) where we are going to store products updated products. It's possible to migrate previously added products from the `products` map to the new storage via migration process however it falls behind the topic of this course.

#### `index.ts`
```javascript=
import { Product, listedProducts } from './model';

export function setProduct(product: Product): void {
    let storedProduct = listedProducts.get(product.id);
    if (storedProduct !== null) { // 1
        throw new Error(`a product with ${product.id} already exists`);
    }
    listedProducts.set(product.id, Product.fromPayload(product));
}

export function getProduct(id: string): Product | null {
    return listedProducts.get(id);
}

export function getProducts(): Product[] {
    return listedProducts.values();
}
```

At line `1` we check if a product with the given id already exists. If it does, we just throw an error so the existing product won't be overridden.

Next, we create a new product based on the argument `product` using the `Product.fromPayload(product)` method.

Also, we added `getProducts()` method that returns all products added.

Here we have made a very important change - we changed `setProduct`'s signature so now it accepts a `product` object instead of two properties - `id` and `productName`. Together with that, we switched to another `PersistentUnorderedMap` - `listedProducts` which now holds `Product` as the value and `listedProducts` (do not forget to import it to `index.ts`).

### Contract redeployment

Now that we changed functions signatures, we would need to deploy a brand-new contract in order to use updated functions. But that's not the case in NEAR.

NEAR API is pretty flexible in terms of introducing changes into an existing smart contract - we can just redeploy it. That's it.

The procedure is exactly the same as the initial deployment - we need to build a `.wasm` file and then deploy using `near deploy` command. However, this time you will be prompted to confirm that action because NEAR already nows that there is a previously deployed contract for a given address.

First, let's build and deploy a smart contract:
```bash=
yarn asb

near deploy --accountId=mycontract.myaccount.testnet --wasmFile=build/release/${WASM_FILE_NAME}
```

Now that we have a `product` object as a parameter we should wrap all product-related properties with an object with `product` key. So `setProduct` call should look like this:
```bash=
near call mycontract.myaccount.testnet setProduct '{"product": {"id": "1", "name": "product_name", "description": "product_description", "location": "image_location", "price": "10000000000"}}' --accountId=myaccount.testnet
```

Let's add some product:
```bash=
near call mycontract.myaccount.testnet setProduct '{"product": {"id": "5", "name": "coffee", "description": "arabica special", "location": "Kenya", "price": "30000000000000000000000000", "image": "coffee.png"}}' --accountId=myaccount.testnet
```

And another invocation of `getProduct(id: string)` to make sure that everything is stored as expected.

```bash=
near view mycontract.myaccount.testnet getProduct '{"id": "5"}'
```

You should see the output like this:
```
View call: mycontract.myaccount.testnet.getProduct({"id": "5"})
{
  id: '5',
  name: 'coffee',
  description: 'arabica special',
  image: 'coffee.png',
  location: 'Kenya',
  price: '30000000000000000000000000',
  owner: 'myaccount.testnet',
  sold: 0
}
```

Now we have two functions that allow us to add and read products, let's add a `buyProduct` function.

### Transactions and transferring tokens

`near-sdk-as` offers an API via `ContractPromiseBatch` to transfer tokens from one account to another.

Before jumping to the details of the API, let's have a closer look at the basics of the transaction.

To complete the transaction, we need to have at least two accounts - a sender account and a receiver account.

Whenever a `change` function is called, there is a `context` object that contains some extra information about the transaction. For instance, it holds a sender account and the attached deposit.

The snipper to transfer token should look like this:
```javascript=
ContractPromiseBatch.create(${RECEIVING_ACCOUNT}).transfer(${DEPOSIT});
```

As we already know, the attached deposit can be taken from the `context` object - `context.attachedDeposit`.

And the receiving account can be taken from the `Product` class where the `owner` property is defined.

Having added this, funds can be transferred to another account. 
Now let's define a `buyProduct` function in the `index.ts` file:

```javascript=
import { ContractPromiseBatch, context } from 'near-sdk-as';

export function buyProduct(productId: string): void {
    const product = getProduct(productId);
    if (product == null) { // 1
        throw new Error("product not found");
    }
    if (product.price.toString() != context.attachedDeposit.toString()) { // 2
        throw new Error("attached deposit should be greater than the product's price");
    }
    ContractPromiseBatch.create(product.owner).transfer(context.attachedDeposit);
    product.incrementSoldAmount(); // 3
    listedProducts.set(product.id, product); // 4
}
```

Here we added a couple of validations:
* `1` - check whether a product by a given `productId` exists
* `2` - check the attached deposit and throw an error if it's not equal to the product's price. 

Also, line `3` calls the` incrementSoldAmount()` function on the product object just to increment the number of sold items.

At line `4` we set the product with the updated `sold` property back to the storage. 

Now we need to redeploy a smart contract.
```bash=
yarn asb

near deploy --accountId=mycontract.myaccount.testnet --wasmFile=build/release/${WASM_FILE_NAME}
```

Let's test `buyProduct` and we would need another account to buy products because it would not be feasible to use a previously created account - we would just buy products from ourselves.

```bash=
near create-account buyeraccount.myaccount.testnet --masterAccount myaccount.testnet --initialBalance 35
```

And now we are ready to buy some products.

```bash=
near call ${CONTRACT_NAME} ${FUNCTION_NAME} ${PAYLOAD} --depositYocto=${ATTACHED_DEPOSIT} --accountId=${BUYER_ACCOUNT}
```
where:
* `--depositYocto` that's a very important parameter that is used to attach yoctoNEAR to the function call so it can be used to transfer to the product's owner. There is another parameter - `--deposit` that we can use to attach NEAR. We use `--depositYocto` here for more convenience so we can just copy the price of a product that `getProduct(id: string)` returns.


Also, in this call `${BUYER_ACCOUNT}` is an id of the account that we created at the previous step.

```bash=
near call mycontract.myaccount.testnet buyProduct '{"productId": "5"}' --depositYocto=30000000000000000000000000 --accountId=buyeraccount.myaccount.testnet
```

If there are no errors, we should see the output like this: 
```
Scheduling a call: mycontract.myaccount.testnet.buyProduct({"productId": "5"}) with attached 30 NEAR
Doing account.functionCall()
Transaction Id {TRANSACTION_ID}
To see the transaction in the transaction explorer, please open this URL in your browser
https://explorer.testnet.near.org/transactions/{TRANSACTION_ID} // 1
''
```

Save the link to the explorer at line `1` - we would need this to check the transaction details.

Now let's verify that the product has been sold successfully.

We will do this in two steps. First, we check that `sold` property of the product was incremented. Last time we read the product it had `0` as a value.

```bash=
near view mycontract.myaccount.testnet getProduct '{"id": "5"}'
```

And the output should look like this:
```
View call: mycontract.myaccount.testnet.getProduct({"id": "5"})
{
  id: '5',
  name: 'coffee',
  description: 'arabica special',
  image: 'coffee.png',
  location: 'Kenya',
  price: '30000000000000000000000000',
  owner: 'myaccount.testnet',
  sold: 1
}
```

`sold` is equal to 1 and it means that we purchased it successfully.

Next, let's follow the link to the explorer from the previous step and verify if the transaction had a correct deposit value and other transaction-related things.

That's it, we have a contract up and running and we can switch to the integration part for a web app.
