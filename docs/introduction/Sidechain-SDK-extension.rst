========================
Sidechains SDK extension
========================

To build a distributed, blockchain application, a developer typically needs to do more than just receive, transfer, and send coins back to the mainchain, as you can do with the basic components provided out-of-the-box by the SDK. Usually, there is the need is to to define some custom data, that the sidechain users can process and exchange according to some defined logic. In this chapter, we'll see what are the steps that should be taken to code a sidechain which implements custom data and logic. In the next one, we'll look in detail at a specific, customized sidechain example.

Custom box creation
###################

The first step of the development process of a distributed app implemented as a sidechain, is the representation of the needed data. In the SDK, application data are modeled as "Boxes". 

Every custom box should at least implement the ``com.horizen.box.Box`` interface. 
The methods defined in the interface are the following:

- ``long nonce()``
  The nonce guarantees that two boxes having the same properties and values, produce different and unique ids.
- ``long value()``
  If the box type is a Coin-Box,  this value is required and will contain the coin value of the Box. 
  In the case of a Non-Coin box, this value is still required, and could have a customized meaning chosen by the developer, or no meaning, i.e. not used. In the latter case, by convention is generally set to 1.
- ``Proposition proposition()``  
  should return the proposition that locks this box.
  The proposition that is used in the SDK examples is `com.horizen.proposition.PublicKey25519Proposition <https://github.com/HorizenOfficial/Sidechains-SDK/blob/master/sdk/src/main/java/com/horizen/proposition/PublicKey25519Proposition.java>`_; it's based on `Curve 25519 <https://en.wikipedia.org/wiki/Curve25519>`_, a fast and secure elliptic curve used by Horizen mainchain. A developer may want to define and use custom propositions.
- ``byte[] id()``
  should return a unique identifier of each box instance.
- ``byte[] bytes()``
  should return the byte representation of this box.
- ``BoxSerializer serializer()``
  should return the serializer of the box (see below).
- ``byte boxTypeId()``
  should return the unique identifier of the box type: each box type must have a unique identifier inside the whole sidechain application.
- ``String typeName()``
  should return the name of class
- ``boolean isCustom()``
should return true for all custom boxes

As a common design rule, you usually do not implement the Box interface directly, but extend instead the abstract class `com.horizen.box.AbstractBox <https://github.com/HorizenOfficial/Sidechains-SDK/blob/master/sdk/src/main/java/com/horizen/box/AbstractBox.java>`_, which already provides default implementations of 
some useful methods like ``id()``, ``equals()``, ``hashCode()``, ``typeName()`` and ``isCustom()``.
This class requires the definition of another object: a class extending `com.horizen.box.AbstractBoxData <https://github.com/HorizenOfficial/Sidechains-SDK/blob/master/sdk/src/main/java/com/horizen/box/AbstractBoxData.java>`_, where you should put all the properties of the box, including the proposition. You can think of the AbstractBoxData as an inner container of all the fields of your box.
This data object must be passed in the constructor of AbstractBox, along with the nonce.
The important methods of AbstractBoxData that need to be implemented are:

- ``byte[] customFieldsHash()``
  Must return a hash of all custom data values, otherwise those data will not be "protected," i.e., some malicious actor can change custom data during transaction creation. 
- ``Box getBox(long nonce)`` 
  creates a new Box containing this BoxData for a given nonce.
- ``BoxDataSerializer serializer()``
  should return the serializer of this box data (see below)

BoxSerializer and BoxDataSerializer
#########################################

Each box must define its own serializer and return it from the ``serializer()`` method.
The serializer is responsible to convert the box into bytes, and parse it back later. It should implement the `com.horizen.box.BoxSerializer <https://github.com/HorizenOfficial/Sidechains-SDK/blob/master/sdk/src/main/java/com/horizen/box/BoxSerializer.java>`_ interface, which defines two methods:

- void ``serialize(Box box, scorex.util.serialization.Writer writer)``
  writes the box content into a Scorex writer  
- Box ``parse(scorex.util.serialization.Reader reader)``
  perform the opposite operation (reads a Scorex reader and re-create the Box)

Also any instance of AbstractBoxData needs to have its own serializer: if you declare a boxData, you should define one in a similar way. In this case the interface to be implemented is `com.horizen.box.data.BoxDataSerializer <https://github.com/HorizenOfficial/Sidechains-SDK/blob/master/sdk/src/main/java/com/horizen/box/data/BoxDataSerializer.java>`_

      
Specific actions for extension of Coin-box
###########################################

A Coin Box is a Box that has a value in ZEN. The creation process is the same just described, with only one extra action: a *Coin box class* needs to implement the ``CoinsBox<P extends PublicKey25519Proposition>`` interface, without the implementation of any additional function (i.e. it's a mixin interface).


Transaction extension
#####################

A transaction is the basic way to implement the application logic, by processing input Boxes that get unlocked and opened (or "spent"), and create new ones. All custom transactions inherited from SidechainTransaction. SidechainNoncedTransaction - class that helps to deal with output boxes nonces. AbstractRegularTransaction class helps to deal with ZenBoxes. To define a new custom transaction, you have to extend the `com.horizen.transaction.SidechainNoncedTransaction <https://github.com/HorizenOfficial/Sidechains-SDK/blob/master/sdk/src/main/java/com/horizen/transaction/SidechainNoncedTransaction.java>`_ class or `com.horizen.transaction.SidechainTransaction <https://github.com/HorizenOfficial/Sidechains-SDK/blob/master/sdk/src/main/java/com/horizen/transaction/SidechainTransaction.java>`_.
The most relevant methods of this class are detailed below:

- ``public List<BoxUnlocker<Proposition>> unlockers()``

  Defines the list of Boxes that are opened when the transaction is executed, together with the information (Proof) needed to open them.
  Each element of the returned list is an instance of BoxUnlocker, which is an interface with two methods:

  ::

    public interface BoxUnlocker<P extends Proposition>
    {
      byte[] closedBoxId();
      Proof<P> boxKey();
    }

  The two methods define the id of the closed box to be opened and the proof that unlocks the proposition for that box. When a box is unlocked and opened, it is spent or "burnt", i.e. it stops existing; as such, it will be removed from the wallet and the blockchain state. As a reminder, a value inside a box cannot be "updated": the process requires to spend the box and create a new one with the updated values.

- ``public List<Box<Proposition>> newBoxes()``

  This function returns the list of new boxes which will be created by the current transaction. 
  As a good practice, you should use the ``Collections.unmodifiableList()`` method to wrap the returned list into a not updatable Collection:

  ::

    @Override
    public List<Box<Proposition>> newBoxes() {
      List<Box<Proposition>> newBoxes =  .....  //new boxes are created here
      //....
      return Collections.unmodifiableList(newBoxes);
    }   

- ``public long fee()``
  Returns the fee to be paid to execute this transaction.

- ``public byte transactionTypeId()``
  Returns the type of this transaction. Each custom transaction must have its own unique type.

- ``public boolean transactionSemanticValidity()``
  Confirms if a transaction is semantically valid, e.g. checks that fee > 0, timestamp > 0, etc.
  This function is not aware of the state of the sidechain, so it can't check, for instance, if the input is a valid Box.

SidechainNoncedTransaction has already implementation of newBoxes function. But it requires an implementation of abstract function getOutputData that provides list of output data of the transaction.
AbstractRegularTransaction requires the implementation of getCustomOutputData for retrieving output custom data of the transaction. The output of other data in AbstractRegularTransaction is already collected in the getOutputData function, which also uses getCustomOutputData.

Apart from the semantic check, the Sidechain will need to make also sure that all transactions are compliant with the application logic and syntax. Such checks need to be implemented in the ``validate()`` method of the ``custom ApplicationState`` class.

Transactions that process Coins
-------------------------------

| A key element of sidechains is the ability to trade ZEN. 
| ZEN are represented as Coin boxes, that can be spent and created. 
Transactions handling coin boxes will generally perform some basic, standard operations, such as: 

- select and collect a list of coin boxes in input which sum up to a value that is equal or higher than the amount to be spent plus fee

- create a coin box with the change

- check that the sum of the input boxes + fee is equal to the sum of the output coin boxes. 

Inside the Lambo-registry demo application, you can find an example of implementation of a transaction that handles regular coin boxes and implements the basic operations just mentioned: `io.horizen.lambo.car.transaction.AbstractRegularTransaction <https://github.com/HorizenOfficial/lambo-registry/blob/master/src/main/java/io/horizen/lambo/car/transaction/AbstractRegularTransaction.java>`_. 
Please note that, in a decentralized environment, transactions generally require the payment of a fee, so that their inclusion in a block can be rewarded and so incentivised. So, even if a transaction is not meant to process coin boxes, it still needs to handle coins to pay its fee.


Custom Proof / Proposition creation
###################################

A proposition is a locker for a box, and a proof is an unlocker for a box. How a box is locked and unlocked can be changed by the developer. For example, a custom box might require to be opened by two or more independent private keys. This kind of customization is achieved by defining custom Proposition and Proof.

* Creating custom Proposition
  You can create a custom proposition by implementing the ``ProofOfKnowledgeProposition<S extends Secret>`` interface. The generic parameter S represents the kind of private key used to unlock the proposition, e.g. you could use *PrivateKey25519*. 
  Let's see how you could declare a new kind of Proposition that accepts two different public keys, and that can be opened by just one of two corresponding private keys:
  ::
    
    public final class MultiProposition implements ProofOfKnowledgeProposition<PrivateKey25519> {
      
      // Specify json attribute name for the firstPublicKeyBytes field.
      @JsonProperty("firstPublicKey")
      private final byte[] firstPublicKeyBytes;

      // Specify json attribute name for the secondPublicKeyBytes field.
      @JsonProperty("secondPublicKey")
      private final byte[] secondPublicKeyBytes;

      public MultiProposition(byte[] firstPublicKeyBytes, byte[] secondPublicKeyBytes) {
          if(firstPublicKeyBytes.length != KEY_LENGTH)
              throw new IllegalArgumentException(String.format("Incorrect firstPublicKeyBytes length, %d expected, %d found", KEY_LENGTH, firstPublicKeyBytes.length));

          if(secondPublicKeyBytes.length != KEY_LENGTH)
              throw new IllegalArgumentException(String.format("Incorrect secondPublicKeyBytes length, %d expected, %d found", KEY_LENGTH, secondPublicKeyBytes.length));

          this.firstPublicKeyBytes = Arrays.copyOf(firstPublicKeyBytes, KEY_LENGTH);
          this.secondPublicKeyBytes = Arrays.copyOf(secondPublicKeyBytes, KEY_LENGTH);
      }

      public  byte[] getFirstPublicKeyBytes() { return firstPublicKeyBytes;}
      public  byte[] getScondPublicKeyBytes() { return secondPublicKeyBytes;}

      //other required methods for serialization omitted here:
      //byte[] bytes()
      //PropositionSerializer serializer();

    }

* Creating custom Proof interface 
  You can create a custom proof by implementing ``Proof<P extends Proposition>``, where *P* is the Proposition class that this Proof can open.
  You also need to implement the ``boolean isValid(P proposition, byte[] messageToVerify);`` function; it checks and states whether Proof is valid for a given Proposition or not. For example, the Proof to open the "two public keys" Proposition shown above could be coded this way:

  ::

    public class MultiSpendingProof extends Proof<MultiProposition> {

          protected final byte[] signatureBytes;

          public MultiSpendingProof(byte[] signatureBytes) {
              this.signatureBytes = Arrays.copyOf(signatureBytes, signatureBytes.length);
          }

          @Override
          public boolean isValid(MultiProposition proposition, byte[] message) {
              return (
                Ed25519.verify(signatureBytes, message, proposition.getFirstPublicKeyBytes()) || 
                Ed25519.verify(signatureBytes, message, proposition.getSecondPublicKeyBytes()
                );
          }

          //other required methods for serialization omitted here:
          //byte[] bytes();
          //ProofSerializer serializer();
          //byte proofTypeId();
    }


Application State
###########################

If we consider the representation of a blockchain in a node as a finite state machine, then the application state can be seen as the state of all the "registers" of the machine at the present moment. The present moment starts when the most recent block is received (or forged!) by the node, and ends when a new one is received/forged. A new block updates the state, so it needs to be checked for both semantic and contextual validity; if ok, the state needs to be updated according to what is in the block.
A customized blockchain will likely include custom data and transactions. The ApplicationState interface needs to be extended to code the rules that state validity of blocks and transactions, and the actions to be performed when a block modifies the state ("onApplyChanges"), and when it is removed ("onRollback", blocks can be reverted!):

ApplicationState:
::
  interface ApplicationState {
      void validate(SidechainStateReader stateReader, SidechainBlock block) throws IllegalArgumentException;

      void validate(SidechainStateReader stateReader, BoxTransaction<Proposition, Box<Proposition>> transaction) throws IllegalArgumentException;

      Try<ApplicationState> onApplyChanges(SidechainStateReader stateReader, byte[] blockId, List<Box<Proposition>> newBoxes, List<byte[]> boxIdsToRemove);

      Try<ApplicationState> onRollback(byte[] blockId);

      boolean checkStoragesVersion(byte[] blockId);

    Try<ApplicationState> onBackupRestore(BoxIterator i);
  }

An example might help to understand the purpose of these methods. Let's assume, as we'll see in the next chapter, that our sidechain can represent a physical car as a token, that is coded as a "CarBox". Each CarBox token should represent a unique car, and that will mean having a unique VIN (Vehicle Identification Number): the sidechain developer will make ApplicationState store the list of all seen VINs, and reject transactions that create CarBox tokens with any preexisting VINs.

Then, the developer could implement the needed custom state checks in the following way:
    ::

      public boolean validate(SidechainStateReader stateReader, BoxTransaction<Proposition, Box<Proposition>> transaction) 

  * Custom checks on transactions should be performed here. If the function throws exception, then the transaction is considered invalid. This method is called either before including a transaction inside the memory pool or before accepting a new block from the network.
    ::

      void validate(SidechainStateReader stateReader, SidechainBlock block) throws IllegalArgumentException
    
  
  * Custom block validation should happen here. If the function throws exception, then the block will not be accepted by the sidechain node. Note that each transaction contained in the block had been already validated by the previous method, so here you should include only block-related checks (e.g. check that two different transactions in the same block don't declare the same VIN car)
    ::

      public boolean validate(SidechainStateReader stateReader, BoxTransaction<Proposition, Box<Proposition>> transaction)

  * Any specific action to be performed after applying the block to the State should be defined here.

    ::

      public Try<ApplicationState> onApplyChanges(SidechainStateReader stateReader, byte[] version, List<Box<Proposition>> newBoxes, List<byte[]> boxIdsToRemove)
    
  
  * Any specific action after a rollback of the state (for example, in case of fork/reverted block) should be defined here.
    ::

      public Try<ApplicationState> onRollback(byte[] version)
    
  
  * This method checks that all the storages of the application which get updated by the sdk via the "onApplyChange" call above, have the version corresponding to the
    blockId passed as input parameter. This is useful when checking the alignment of sdk and application storages versions at node restart.
    ::

      public boolean checkStoragesVersion(byte[] blockId)
    
  
  * This method is used during the restore procedure. It can be useful if you want to perform some operations based on the restored boxes.
    ::

      Try<ApplicationState> onBackupRestore(BoxIterator i);

Application Wallet 
##################

Every sidechain node has a local wallet associated to it, in a similar way as the mainchain Zend node wallet.
The wallet stores the user secret info and related balances. It is initialized with the genesis account key and the ZEN amount transferred by the sidechain creation transaction.
New private keys can be added by calling the http endpoint /wallet/createPrivateKey25519.
The local wallet data is updated when a new block is added to the sidechain, and when blocks are reverted. 

Developers can extend Wallet logic by defining a class that implements the interface `ApplicationWallet <https://github.com/ZencashOfficial/Sidechains-SDK/blob/master/sdk/src/main/java/com/horizen/wallet/ApplicationWallet.java>`_
The interface methods are listed below:

::

  interface ApplicationWallet {
      void onAddSecret(Secret secret);

      void onRemoveSecret(Proposition proposition);

      void onChangeBoxes(byte[] version, List<Box<Proposition>> boxesToBeAdded, List<byte[]> boxIdsToRemove);

      void onRollback(byte[] version);

      boolean checkStoragesVersion(byte[] blockId);

      void onBackupRestore(BoxIterator i);
  }

As an example, the onChangeBoxes method gets called every time new blocks are added or removed from the chain; it can be used to implement for instance the update to a local storage of values that are modified by the opening and/or creation of specific box types.
Similarly to ApplicationState, the checkStoragesVersion method is useful when checking the alignment of sdk and application wallet storages versions at node restart.



Sidechain Application Stopper 
#############################

A user application should define a class that implements the interface FIXME `SidechainAppStopper <https://github.com/ZencashOfficial/Sidechains-SDK/blob/master/sdk/src/main/java/com/horizen/SidechainAppStopper.java>`_
The interface is listed below:

::

  interface SidechainAppStopper {
      void stopAll();
  }

The stopAll() method gets called when the node stop procedure is initiated. Such a procedure can be explicitly triggered via the API ‘node/stop’ or can be triggered when the JVM is shutting down, for instance when a SIGINT is received.
In the custom implementation for instance, custom storages should be closed and any resources should be properly released. An example is provided in the “SimpleApp” with the SimpleAppStopper.java class.


Custom API creation 
###################

A user application can extend the default standard API (see chapter 6) and add custom API endpoints. For example if your application defines a custom transaction, you may want to add an endpoint that creates one.

To add custom API you have to create a class which extends the com.horizen.api.http.ApplicationApiGroup abstract class, and implements the following methods:

-  ``public String basePath()``
   returns the base path of this group of endpoints (the first part of the URL)

-  ``public List<Route> getRoutes()``
   returns a list of Route objects: each one is an instance of a `akka.Http Route object <https://doc.akka.io/docs/akka-http/current/routing-dsl/routes.html>`_ and defines a specific endpoint url and its logic.
   To simplify the developement, the ApplicationApiGroup abstract class provides a method (bindPostRequest) that builds a akka Route that responds to a specific http request with an (optional) json body as input. This method receives the following parameters:
   
   - the endpoint path

   - the function to process the request 

   - the class that represents the input data received by the  HTTP request call 
   
   Example:
    ::

      public List<Route> getRoutes() {
            List<Route> routes = new ArrayList<>();
            routes.add(bindPostRequest("createCar", this::createCar, CreateCarBoxRequest.class));
            routes.add(bindPostRequest("createCarSellOrder", this::createCarSellOrder, CreateCarSellOrderRequest.class));
            routes.add(bindPostRequest("acceptCarSellOrder", this::acceptCarSellOrder, SpendCarSellOrderRequest.class));
            routes.add(bindPostRequest("cancelCarSellOrder", this::cancelCarSellOrder, SpendCarSellOrderRequest.class));
            return routes;
        }

    Let's look in more details at the 3 parameters of the bindPostRequest method.

    - The endpoint path: 
      defines the endpoint path, that appended to the basePath will represent the http endpoint url.
       | For example, if your API group has a basepath = "carApi", and you define a route with endpoint path "createCar", the overall url will be: ``http://<node_host>:<api_port>/carAPi/createCar``

    - The function to process the request:
      Currently we support three types of function’s signature:
    
        * ApiResponse ``custom_function_name(Custom_HTTP_request_type)`` -- a function that by default does not have access to *SidechainNodeView*. 

        * ``ApiResponse custom_function_name(SidechainNodeView, Custom_HTTP_request_type)`` -- a function that offers by default access to SidechainNodeView
        
        * ``ApiResponse custom_function_name(SidechainNodeView)`` -- a function to process empty HTTP requests, i.e. endpoints that can be called without a JSON body in the request

        The format of the ApiResponse to be returned will be described later in this chapter.

    - The class that represents the body in the HTTP request
      
      | This needs to be a java bean, defining some private fields and  getter and setter methods for each field.
      | Each field in the json input will be mapped to the corresponding field by name-matching.
      | For example to handle the  following json body :
      ::
        
        {
        "number": "342",
        "someBytes": "a5b10622d70f094b7276e04608d97c7c699c8700164f78e16fe5e8082f4bb2ac"
        }

      you should code a request class like this one:
      ::

        public class MyCustomRequest {
          byte[] someBytes;
          int number;

          public byte[] getSomeBytes(){
            return someBytes;
          }

          public void setSomeBytes(String bytesInHex){
            someBytes = BytesUtils.fromHexString(bytesInHex);
          }

          public int getNumber(){
            return number;
          }

          public void setNumber(int number){
            this.number = number;
          }
        }

Submitting transaction can be operated with TransactionSubmitProvider
::
    trait TransactionSubmitProvider {
       @throws(classOf[IllegalArgumentException])
       def submitTransaction(tx: BoxTransaction[Proposition, Box[Proposition]]): Unit

       def asyncSubmitTransaction(tx: BoxTransaction[Proposition, Box[Proposition]],
                                  callback:(Boolean, Option[Throwable]) => Unit): Unit
    }

For example
::
    val transactionSubmitProvider: TransactionSubmitProviderImpl = new TransactionSubmitProviderImpl(sidechainTransactionActorRef)

    val tryRes: Try[Unit] = Try {
      transactionSubmitProvider.submitTransaction(transaction)
    }
    tryRes match {
      case Success(_) => // expected behavior
      case Failure(exception) => fail("Transaction expected to be submitted successfully.", exception)
    }

asyncSubmitTransaction allows after submitting transaction apply callback function.

::

    val transactionSubmitProvider: TransactionSubmitProviderImpl = new TransactionSubmitProviderImpl(sidechainTransactionActorRef)


    def callback(res: Boolean, errorOpt: Option[Throwable]): Unit = synchronized {
        // Some operations executed after submitting transaction
    }

    // Start submission operation ...
    transactionSubmitProvider.asyncSubmitTransaction(transaction, callback)


Also there are available providers for retrieving NodeView and Secret submission

::

    trait NodeViewProvider {
        def getNodeView(view: SidechainNodeView => Unit)

    }

::

    public interface SecretSubmitHelper {
        void submitSecret(Secret secret) throws IllegalArgumentException;
    }

API response classes

The function that processes the request must return an object of type com.horizen.api.http.ApiResponse.
In most cases, we can have two different responses: either the operation is successful, or an error has occurred during the API request processing. 

For a successful response, you have to:
- define an object implementing the  SuccessResponse interface
- add the annotation  @JsonView(Views.Default.class) on top of the class, to allow the automatic conversion of the object into a json format.
- add some getters representing the values you want to return.

 For example, if a string should be returned, then the following response class can be defined:

  ::
  
    @JsonView(Views.Default.class)
    class CustomSuccessResponce implements SuccessResponse{
      private final String response;

      public CustomSuccessResponce (String response) {
        this.response = response;
      }

      public String getResponse() {
        return response;
      }
    }

In such a case, the API response will be represented in the following JSON format:

  ::
  
    {"result": {“response” : “response from CustomSuccessResponse object”}}


    
If an error is returned, then the response will implement the ErrorResponse interface. The ErrorResponse interface has the following default functions implemented:

```public String code()``` -- error code

```public String description()``` -- error description 

```public Option<Throwable> exception()``` -- Caught exception during API processing

As a result the following JSON will be returned in case of error:

  ::
  
    {
      "error": 
      {
      "code": "Defined error code",
      "description": "Defined error description",
      "Detail": “Exception stack trace”
      }
    }

  
Custom api group injection:

Finally, you have to instruct the SDK to use your ApiGroup.
This can be done with Guice, by binding the ""CustomApiGroups" field:
::

   bind(new TypeLiteral<List<ApplicationApiGroup>> () {})
         .annotatedWith(Names.named("CustomApiGroups"))
         .toInstance(mycustomApiGroups);

.. _backup_and_restore-label:

Backup and restore procedure
###################

This mechanism was introduced to make it possibile to bootstrap a sidechain starting from a "snapshot" taken from an another sidechain of the same kind.
This can be useful in the unfortunately case of a sidechain that get ceased. With this procedure you are able to make a backup of your unspent non-coin boxes contained
in your ceased sidechain and start a new sidechain that contains these boxes.

Important notes:
    ::

- This procedure allows to backup and restore only NON-COIN boxes.
- These restored boxes are not propagated over the network. This means also that, in a re-bootstrapped sidechain, every single node must have these boxes inside it's own data directory.
- The nodes must include the backup inside their data directory BEFORE they are started for the first time.

Backup procedure
-------------------------------

The SDK contains a Class called ``SidechainAppBackup`` that can be referenced from the application level to perform a backup.

::

    class SidechainBackup @Inject()
      (@Named("CustomBoxSerializers") val customBoxSerializers: JHashMap[JByte, BoxSerializer[SidechainTypes#SCB]],
       @Named("BackupStorage") val backUpStorage: Storage,
       @Named("BackUpper") val backUpper : BoxBackupInterface
      ) extends ScorexLogging
      {

        def createBackup(stateStoragePath: String, sidechainBlockIdToRollback: String, copyStateStorage: Boolean): Unit = {
            ...
        }
      }

It requires that the application level injects the following objects:

- ``CustomBoxSerializer``: Map containing a serializer for every kind of new boxes added.
- ``BackupStorage``: The Storage that will contains the backup.
- ``BackUpper``: A class that implements the interface ``BoxBackupInterface`` that will be called to perform the backup.

The method ``createBackup`` starts the backup procedure by calling the function ``BackUpper.backup``. It takes as parameters:

- ``stateStoragePath``: File path to the SidechainStateStorage.
- ``sidechainBlockItToRollback``: Sidechain block id used for the storage rollback.
- ``copyStateStorage``: If True performs a copy of the SidechainStateStorage before rollback it in order to avoid its permanent corruption (this is a one way procedure, the Storge can't be used anymore after the rollback).

The ``BoxBackupInterface`` interface declares a method ``backup`` that should be implemented by the extending class.

::

    public interface BoxBackupInterface {
        void backup(BoxIterator source, BackupStorage db) throws Exception;
    }

The ``backup`` method receives an iterator over the Storage that will be taken as a source for the backup (typically it would be the SidechainStateStorage),
and the Storage used to store the backup.
You can use the method ``nextBox`` from this iterator to retrieve the next non-coin box from the Storage.

Important notes:
    ::

In order to maintain a consistency between what the Mainchain knows about the Sidechain and what the Sidechain contains itself, if you want to perform a Backup of a ceased Sidechain, you should rollback the Storage
to the version that contains the Mainchain block calculated by the following formula:

  Genesis_MC_block_height + (current_epch - 2) * withdrawalEpochLength - 1

This can be easily done by calling the endpoint `/backup/getSidechainBlockIdForBackup <https://github.com/HorizenOfficial/Sidechains-SDK/blob/master/sdk/src/main/scala/com/horizen/api/http/SidechainBackupApiRoute.scala>`_
and pass the block id obtained to the method ``createBackup``.

Restore procedure
-------------------------------

The restore procedure is automatically invoked when a Sidechain node starts from an empty blockchain. Before the application of the genesis block, the node is able to detect if there is a Backup Storage to restore
into it's data directory; in such a case, it performs several iterations over it in order to populate the other storages.
The Backup Storage is scanned 4 times:

- First scan used to populate the ``SidechainStateStorage`` with all the boxes found in the BackupStorage.
- Second scan used to add the boxes owned by a node wallet proposition inside the ``WalletBoxStorage``. In this way you will be able to see the restored boxes inside your wallet (only if you have the corresponding proposition imported in your wallet) and spend them.
- Third scan performed in the application level (``ApplicationState``). This is useful if you have some custom storages that you want to populate with the information taken from these boxes. You should override the method ``public Try<ApplicationState> onBackupRestore(BoxIterator boxIterator)`` inside the ``ApplicationState``.
- Fourth scan performed in the application level (``ApplicationWallet``). This is useful if you want to perform some operations in your wallet based on the information taken from these boxes. You should override the method ``public void onBackupRestore(BoxIterator i)`` inside the ``ApplicationWallet``.

Important notes:
    ::

The Backup Storage must be present inside your node data directory before starts the node for the first time.

The procedure fails if just a single coin box is found inside the Backup Storage.

If you own some of the restored boxes and you want to see them inside your wallet, you should add your secrets inside the config file of your node (before start the node for the first time).
You can add your secrets inside the section Wallet.genesisSecrets by appending "00" at the beginning of your secret in case it is a PrivateKey25519, "03" if it is a VrfPrivateKey or "04" if it is a SchnorrPrivateKey.

Logging
###################

The SDK logging system is based on the Log4J library.
To fire additional log messages in your application code, just declare a log4j Logger in your class and use it:

::

    public class MyCustomClass {

        Logger logger = LogManager.getLogger(MyCustomClass.class);

        public MyCustomClass(){
          logger.debug("This is an example  debug message inside the constructor");
        }
    }

Note: do not add any log4j library in your application pom file, as it is already loaded as a nested dependency of the SDK. 

You can rely on the default logging configuration (based on Log4J library) and change just a few parameters inside the application configuration file, or override it completely with a custom one.

Default log configuration
-------------------------------
By default, `a predefined log4j2.xml configuration <https://github.com/HorizenOfficial/Sidechains-SDK/blob/master/sdk/src/main/resources/log4j2.xml>`_  is used.
It redirects all logging messages to the system console and to a filesystem log file, rotated and gzipped when it reaches 50MB size (only the latest 10 are then retained).

The following dynamic parameters are taken from the application configuration file, and  can be changed there at any time:
::

    scorex {
        ...

        logDir = /tmp/scorex/data/log

        logInfo {
          logFileName = "debug.log"
          logFileLevel = "warn"
          logConsoleLevel = "debug"
        }

        ...

- ``scorex.logDir``: base folder where the log files are generated, injected in the log4jxml in the placeholder ``${sys:logDir}``
- ``scorex.logInfo.logFileName`` : log filename, injected in the log4jxml in the placeholder ``${sys:logFileName}``
- ``scorex.logInfo.logFileLevel`` : log level used for the file appender, injected in the log4jxml in the placeholder ``${sys:logFileLevel}``
- ``scorex.logInfo.logConsoleLevel`` : log level used for the console appender, injected in the log4jxml in the placeholder ``${sys:logConsoleLevel}``

Customized log configuration
-------------------------------

If you add a custom log4j2.xml in your application's classpath, it will override the default one.

Same placeholders described before are available also here.

Fork Configuration
###################

In the new versions of the SDK the backward incompatible changes may be introduced. For example, the changes to the consensus protocol or another kind of core transaction.
That will lead to the hard fork. So, already running sidechains must be very careful and may have a unified mechanism to upgrade the nodes and activate such changes.
For this every sidechain application should use `ForkConfigurator` to specify the activation points for regtest, testnet and mainnet networks.
SDK implements consensus, which is measured in the fixed time epochs, but an epoch may have variant number of blocks associated with it.
That's why forks activation points (caused by consensus changes) are specified in the consensus epoch numbers, rather than blockchain heights.
Every sidechain network knows what is the current epoch and able to choose any epoch in the future for the fork activation while upgrading SDK version.

In the following example the first fork activates in regtest at epoch 10, in testnet at epoch 20, and in mainnet at epoch 30.
The next fork is activated always not earlier than the previous one (but can be activated at the same time).

::

  public class MyAppForkConfigurator extends ForkConfigurator {
      @Override
      public ForkConsensusEpochNumber getSidechainFork1() {
          return new ForkConsensusEpochNumber(/*regtest*/ 10, /*testnet*/ 20, /*mainnet*/30);
      }

      @Override
      public ForkConsensusEpochNumber getSidechainFork2() {
          return new ForkConsensusEpochNumber(/*regtest*/ 20, /*testnet*/ 20, /*mainnet*/30);
      }
  }

