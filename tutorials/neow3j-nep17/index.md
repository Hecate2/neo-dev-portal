---
title: 'neow3j - Implementing a NEP-17 Smart Contract in Java'
description: "This tutorial describes an example implementation of a NEP-17 smart contract developed in Java using the neow3j library."
author: AxLabs
tags: ["NEP-17", "JAVA", "NEOW3J"]
skill: beginner
image: "./assets/neow3j-padded.png"
source: "https://github.com/neow3j/neow3j-examples-java/blob/ddd90914ea4ec5f928066a582012043bbce01525/src/main/java/io/neow3j/examples/contractdevelopment/contracts/FungibleToken.java"
sidebar: true
---

<div align="center" style={{ padding: '0% 25% 0% 25%' }}>
  <img src="/tooling/neow3j.png" alt="neow3j" width="75%" style={{ padding: '0% 0% 5% 0%' }}/> 
  <h1> <a href="https://github.com/neow3j/neow3j">neow3j</a> <sub><small>v3.16.0</small></sub></h1> 
</div>

Neow3j is a development toolkit that provides easy and reliable tools to build Neo dApps and Smart
Contracts using the Java platform (Java, Kotlin, Android). Check out [neow3j.io](https://neow3j.io) for more detailed information on neow3j and the technical documentation.

## 1. Setup

If you haven't already set up your environment to use the neow3j library, you can check out our tutorial about setting up a neow3j project [here](/tutorials/neow3j-smart-contract-quickstart).

## 2. NEP-17 Overview

The NEP-17 is the fungible token standard on Neo N3. Have a look at its official documentation [here](https://github.com/neo-project/proposals/blob/master/nep-17.mediawiki).

## 3. Example NEP-17 Contract

The following example code represents a possible implementation for a token that supports the NEP-17 standard.

```java
package io.neow3j.examples.contractdevelopment.contracts;

import static io.neow3j.devpack.StringLiteralHelper.addressToScriptHash;

import io.neow3j.devpack.ByteString;
import io.neow3j.devpack.Contract;
import io.neow3j.devpack.Hash160;
import io.neow3j.devpack.Runtime;
import io.neow3j.devpack.Storage;
import io.neow3j.devpack.StorageContext;
import io.neow3j.devpack.StorageMap;
import io.neow3j.devpack.annotations.DisplayName;
import io.neow3j.devpack.annotations.ManifestExtra;
import io.neow3j.devpack.annotations.OnDeployment;
import io.neow3j.devpack.annotations.Permission;
import io.neow3j.devpack.annotations.Safe;
import io.neow3j.devpack.annotations.SupportedStandard;
import io.neow3j.devpack.constants.CallFlags;
import io.neow3j.devpack.constants.NativeContract;
import io.neow3j.devpack.constants.NeoStandard;
import io.neow3j.devpack.contracts.ContractManagement;
import io.neow3j.devpack.events.Event3Args;

@DisplayName("AxLabsToken")
@ManifestExtra(key = "author", value = "AxLabs")
@SupportedStandard(neoStandard = NeoStandard.NEP_17)
@Permission(nativeContract = NativeContract.ContractManagement)
public class FungibleToken {

    // Constants

    static final Hash160 owner = addressToScriptHash("NM7Aky765FG8NhhwtxjXRx7jEL1cnw7PBP");

    static final int decimals = 2;
    static final byte[] totalSupplyKey = new byte[]{0x00};
    static final StorageContext ctx = Storage.getStorageContext();
    static final StorageMap assetMap = new StorageMap(ctx, new byte[]{0x01});

    // NEP-17 Events

    @DisplayName("Transfer")
    static Event3Args<Hash160, Hash160, Integer> onTransfer;

    // NEP-17 Methods

    @Safe
    public static String symbol() {
        return "ALT";
    }

    @Safe
    public static int decimals() {
        return decimals;
    }

    @Safe
    public static int totalSupply() {
        return Storage.getInt(ctx, totalSupplyKey);
    }

    @Safe
    public static int balanceOf(Hash160 account) {
        assert Hash160.isValid(account) : "Argument is not a valid address.";
        return getBalance(account);
    }

    public static boolean transfer(Hash160 from, Hash160 to, int amount, Object[] data) {

        assert Hash160.isValid(from) && Hash160.isValid(to) : "'from' or 'to' address is not a valid address.";
        assert amount >= 0 : "The transfer amount must be non-negative.";
        assert Runtime.checkWitness(from) : "No authorization.";

        if (amount > getBalance(from)) {
            return false;
        }

        if (from != to && amount != 0) {
            deductFromBalance(from, amount);
            addToBalance(to, amount);
        }
        if (ContractManagement.getContract(to) != null) {
            onTransfer.fire(from, to, amount);
            Contract.call(to, "onNEP17Payment", CallFlags.All, data);
        }
        return true;
    }

    // Private Helper Methods

    private static void throwIfSignerIsNotOwner() {
        assert Runtime.checkWitness(owner) : "The calling entity is not the owner of this contract.";
    }

    private static void addToBalance(Hash160 key, int value) {
        assetMap.put(key.toByteArray(), getBalance(key) + value);
    }

    private static void deductFromBalance(Hash160 key, int value) {
        int oldValue = getBalance(key);
        assetMap.put(key.toByteArray(), oldValue - value);
    }

    private static int getBalance(Hash160 key) {
        return assetMap.getIntOrZero(key.toByteArray());
    }

    // Deploy, Update, and Destroy

    @OnDeployment
    public static void deploy(Object data, boolean update) {
        if (!update) {
            throwIfSignerIsNotOwner();
            int initialSupply = 200_000_000;
            Storage.put(ctx, totalSupplyKey, initialSupply);
            assetMap.put(owner.toByteArray(), initialSupply);
        }
    }

    public static void update(ByteString script, String manifest) {
        throwIfSignerIsNotOwner();
        ContractManagement.update(script, manifest);
    }

    public static void destroy() throws Exception {
        throwIfSignerIsNotOwner();
        ContractManagement.destroy();
    }

}
```

## 4. Contract Breakdown

In the following subsections, we'll be looking at each part of the NEP-17 example contract.

### Imports

The imports show the neow3j devpack classes that are used in the example contract. Check out neow3j devpack's [javadoc](https://javadoc.io/doc/io.neow3j/devpack/latest/index.html) for a full overview of classes and methods that are supported.

```java
package io.neow3j.examples.contractdevelopment.contracts;

import static io.neow3j.devpack.StringLiteralHelper.addressToScriptHash;

import io.neow3j.devpack.ByteString;
import io.neow3j.devpack.Contract;
import io.neow3j.devpack.Hash160;
import io.neow3j.devpack.Runtime;
import io.neow3j.devpack.Storage;
import io.neow3j.devpack.StorageContext;
import io.neow3j.devpack.StorageMap;
import io.neow3j.devpack.annotations.DisplayName;
import io.neow3j.devpack.annotations.ManifestExtra;
import io.neow3j.devpack.annotations.OnDeployment;
import io.neow3j.devpack.annotations.Permission;
import io.neow3j.devpack.annotations.Safe;
import io.neow3j.devpack.annotations.SupportedStandard;
import io.neow3j.devpack.constants.CallFlags;
import io.neow3j.devpack.constants.NativeContract;
import io.neow3j.devpack.constants.NeoStandard;
import io.neow3j.devpack.contracts.ContractManagement;
import io.neow3j.devpack.events.Event3Args;
```

### Contract-specific Information

Annotations on top of the smart contract's class represent contract-specific information. The following annotations can be used for in a smart contract:

_`@DisplayName`_

It specifies the contract's name. If this annotation is not present, the class name is used for the contract's name.

_`@ManifestExtra`_

Adds the provided key-value pair information in the manifest's `extra` field. You can also use `@ManifestsExtras` to gather multiple `@ManifestExtra` annotations (results in the same as when using single `@ManifestExtra` annotations).

_`@SupportedStandard`_

Sets the `supportedStandards` field in the manifest. You can use `neoStandard = ` with the enum `NeoStandard` to use an official standard (see [here](https://github.com/neo-project/proposals#readme)), or `customStandard = ` with a custom string value.

_`@Permission`_

Specifies, which third-party contracts and methods the smart contract is allowed to call. By default (i.e., if no permission annotation is set), the contract is not allowed to call any contract. Use `contract = ` and `methods = ` to specify, respectively, which contracts and methods are allowed.

_For example, if you want to allow transferring NEO tokens from the contract, you can add the annotation `@Permission(nativeContract = NativeContract.NeoToken, methods = "transfer")`._

```java
@DisplayName("AxLabsToken")
@ManifestExtra(key = "author", value = "AxLabsToken")
@SupportedStandard(neoStandard = NeoStandard.NEP_17)
@Permission(nativeContract = NativeContract.ContractManagement)
public class FungibleToken {
```

### Constants

You can set a constant value for the contract by using `final` variables. These values are always loaded when the contract is called and cannot be changed once the contract is deployed.

:::note

All contract constants and all methods must be `static` (since the object-orientation of the JVM is different on the NeoVM).

:::
:::tip

The contract owner of this example contract is fixed (i.e., it is a `final` variable). If you intend to provide a way to change such a variable, you should not store it as a `final` variable. Rather, you would store it as a value in the storage, which provides the possibility to be modified through a method.

:::

```java
static final Hash160 owner = addressToScriptHash("NM7Aky765FG8NhhwtxjXRx7jEL1cnw7PBP");

static final int decimals = 2;
static final byte[] totalSupplyKey = new byte[]{0x00};
static final StorageContext ctx = Storage.getStorageContext();
static final StorageMap assetMap = new StorageMap(ctx, new byte[]{0x01});
```

### NEP-17 Methods

The required NEP-17 methods are implemented as follows. If a method does not change the state of the contract (i.e., it is just used for reading), it can be annotated with the `@Safe` annotation. Out of the NEP-17 methods, only the `transfer()` method should be writing to the contract and is thus not annotated as safe.

:::info

Neow3j uses Java's `assert` statement to throw exceptions on the NeoVM. Thus, there is no need of using an `if` statement with a followed `throw new Exception()` statement. Although, it may still be used like that. It is up to the developer to go either way, as it results in the exact same NeoVM code once compiled.

:::

```java
@Safe
public static String symbol() {
    return "ALT";
}

@Safe
public static int decimals() {
    return decimals;
}

@Safe
public static int totalSupply() {
    return Storage.getInt(ctx, totalSupplyKey);
}

@Safe
public static int balanceOf(Hash160 account) {
    assert Hash160.isValid(account) : "Argument is not a valid address.";
    return getBalance(account);
}

public static boolean transfer(Hash160 from, Hash160 to, int amount, Object[] data) {
    assert Hash160.isValid(from) && Hash160.isValid(to) : "'from' or 'to' address is not a valid address.";
    assert amount >= 0 : "The transfer amount must be non-negative.";
    assert Runtime.checkWitness(from) : "No authorization.";

    if (getBalance(from) < amount) {
        return false;
    }

    if (from != to && amount != 0) {
        deductFromBalance(from, amount);
        addToBalance(to, amount);
    }
    if (ContractManagement.getContract(to) != null) {
        onTransfer.fire(from, to, amount);
        Contract.call(to, "onNEP17Payment", CallFlags.All, data);
    }
    return true;
}
```

### Events

The NEP-17 standard requires an event `Transfer` that contains the values `from`, `to`, and `amount`. For this, the class `Event3Args` can be used with the annotation `@DisplayName` to set the event's name that will be shown in the manifest and notifications when it has been fired.

```java
@DisplayName("Transfer")
static Event3Args<Hash160, Hash160, Integer> onTransfer;
```

An event variable can effectively fire an event by using the `fire()` method with the corresponding arguments. For example, the `Transfer` event (represented by the `onTransfer` variable) should be fired whenever a transfer happens.

```java
onTransfer.fire(from, to, amount);
```

### Deploy

Once a deployment transaction is made (containing the contract and other parameters), the contract data is first stored on the blockchain and then the native contract `ContractManagement` calls the smart contract's `deploy()` method. In neow3j, that method is marked with the annotation `@OnDeployment`. In the example, when the smart contract is deployed, the `initialSupply` is set to 200'000'000 and it is allocated to the smart contract's owner.

```java
@OnDeployment
public static void deploy(Object data, boolean update) {
    if (!update) {
        throwIfSignerIsNotOwner();
        int initialSupply = 200_000_000;
        Storage.put(ctx, totalSupplyKey, initialSupply);
        assetMap.put(owner.toByteArray(), initialSupply);
    }
}
```

### Update and Destroy

In order to update the contract, the following method first checks that the contract owner witnessed the transaction and then the native `ContractManagement.update()` method is called. When updating a smart contract, you can change the smart contract's code and its manifest. This means that you can update how the contract programmatically manages its storage context.

:::note

Additionally to changing the smart contract's script and manifest, the method `ContractManagement.update()` eventually calls the smart contract's `deploy()` method (shown above) with the boolean `update` set to true.

:::

```java
public static void update(ByteString nef, String manifest) {
    throwIfSignerIsNotOwner();
    ContractManagement.update(nef, manifest);
}
```

The example contract also provides the option to destroy the smart contract. As well as the `update()` method, it first verifies that the contract owner witnessed the transaction and then calls the method `ContractManagement.destroy()` method.

:::caution

When the native method `ContractManagement.destroy()` is called from a smart contract, the whole smart contract's storage context is erased, and the contract can no longer be used.

:::

```java
public static void destroy() throws Exception {
    throwIfSignerIsNotOwner();
    ContractManagement.destroy();
}
```

### Private Helper Methods

Private methods can be used to, for example, simplify and make the smart contract a bit more readable. The following private methods are used in the NEP-17 example contract. For example, in order to prevent writing the same exception message to check the witness of the owner, the private method `throwIfSignerIsNotOwner()` allows writing this code only once.

```java
private static void throwIfSignerIsNotOwner() {
    assert Runtime.checkWitness(owner) : "The calling entity is not the owner of this contract.";
}

private static void addToBalance(Hash160 key, int value) {
    assetMap.put(key.toByteArray(), getBalance(key) + value);
}

private static void deductFromBalance(Hash160 key, int value) {
    int oldValue = getBalance(key);
    assetMap.put(key.toByteArray(), oldValue - value);
}

private static int getBalance(Hash160 key) {
    return assetMap.getIntOrZero(key.toByteArray());
}
```

## 5. Compile the Contract

The contract can be compiled using the gradle plugin. First, set the `className` in the file `gradle.build` to the contract's class name. Then, the gradle task `neow3jCompile` can be executed from the project's root path to compile the contract.

```bash
./gradlew neow3jCompile
```

The output is then accessible in the folder `./build/neow3j`, and should contain the following three files:

```bash
AxLabsToken.manifest.json
AxLabsToken.nef
AxLabsToken.nefdbgnfo
```

:::note

The filenames can deviate according to what the contract's name is. See [here](#contract-specific-information).

:::

Now, the contract's `.manifest.json` and `.nef` files can be used to deploy the contract. Neow3j's SDK can be used to do so. Check out the example [here](https://github.com/neow3j/neow3j-examples-java/blob/4d82df91c27bf9d4992c166e1ae98045bd24fbbd/src/main/java/io/neow3j/examples/contractdevelopment/DeployFromFiles.java) about how to deploy a contract with its manifest and nef files.

## About

Feel free to report any issues that might arise. Open an issue [here](https://github.com/neow3j/neow3j/issues/new/choose) to help us directly including it in our backlog.


<!---
## How to test my dApp

This could also be handled in another tutorial
--->
