## Inspiration

We (Aram and Aaron) faced a very simple problem. Aaron really likes biking and invited Aram to go on a bike ride. Problem is, Aram doesn't have a bike and Aaron didn't have one to spare. Aram proceeded to message several of his friends to borrow a bike, eventually finding someone willing and able to share.

Though simple, this **peer to peer sharing** relies on existing trust between Aram and his friends: his friends need to feel assured that he won't just run off with their bike. However, this **trust is hard to scale** past his immediate group of friends.

This specific problem inspired the following observation: hardware secured using chainlink can leverage the guaranteed trust of blockchain apps to create a peer to peer lending platform for "things". **Trust can be assured and scaled using this technology** to connect distant individuals in a mutually beneficial way. 

From golf clubs and bikes, to laptops and vacuums, this platform can create a trusted way for owners to rent out their valuable things to those who want them. Think rental smart contracts (which implement rules like late fees, distance boundaries, and damage clauses) that connect to sensors to automatically execute the clauses of the contract. In short, **the AirBnB for all "things"!**

However, this is a hard problem to entirely tackle in 3 weeks. Being avid readers, we saw **books as a small scale version of this larger vision** that we could get off the ground quickly. We also realized that our tech could leverage the existing physical infrastructure of [Little Free Library](https://littlefreelibrary.org/). Hence, the final version of our idea:

Our idea: a **book rental platform** where the **end UX would be a familiar check in/out system**, but **the rental backend would be decentralized by smart contracts and RFID scanners** enabled by chainlink and ethereum. Physical rental stations would be set up at existing Little Free Library locations, and books can be freely moved between stations. I.e. **a book checked out in San Francisco could be checked in in New York**.

*Note*: this last point about inter-library sharing is quite important. Assuming the product form factor evolves to something simple enough where spreading to new locations does not rely on a physical scanner (perhaps replaced by a QR code and phone app), very low operating costs can translate into **exponential growth as books circulate through the existing Little Free Library network**. 


## What it does

TLDR: **a book check in/check out system; accepts book contributions to the platform from users.** Late fees automatically charged and sent to original contributors (not currently implemented, but easy feature to add). **Incentivizes quality book contributions and responsible lending** (a unique advantage over the current Little Free Library).

For a Little Free Library location, there will be an RFID reader connected to our platform.

Using our website, readers can check out books by scanning them. This initiates a rental smart contract with a fixed term and late fees. Once the book is scanned and checked in again, the rental contract will self destruct and the book will be available to check out from the library again.

Readers can also contribute books to the library by adding a supported RFID tag and scanning the book into the library. They will receive all fees generated by the book. Unlike the original Little Free Library project, this incentives the sharing of higher quality books that will generate revenue for the contributor.

## How we built it

Our tech stack roughly broke down into the following categories (ordered by module dependencies):

- RFID hardware (C/arduino)
- RFID parser/adapter (python) 
- Chainlink node
- ETH smart contracts (solidity) 
- Frontend (HTML/JS/CSS).

### RFID stuff

**Hardware/firmware**: MFRC522 RFID reader connected to an arduino board. Used an [existing](https://github.com/miguelbalboa/rfid) RFID firmware library for the RFID reader.

**Parser**: we wrote a custom state machine to parse the UID from RFID scans. That code is in `rfid-interface/rfid.py`.

**External adapter**: in the same python script, there is also a flask server in `rfid-interface/app.py` that accepts POST requests to get a scan. Upon a request, the scanner will turn on and wait till the next scan or timeout. The UID of the scanned card is then returned.

### Chainlink/smart contract stuff

We opted to run our own chainlink node on the kovan testnet. This connected to the external adapter server running in the above section.

- *Oracle address*: `0x315FF29aC038d62e1AB915081281Eeb0FE503738`
- *Job ID*: `b4103c96c6514e779a9bb386b7f84011`

We wrote 2 custom ethereum smart contracts:

- `contracts/Library.sol`: main contract for the platform. Inherits from `ChainlinkClient` to connect to the RFID scanner. Primary interface functions:	
	- `mapping(bytes32 ==> Book) bookCollection`: maps rfid UID tags to book structs (collection of relevant data about a single entry in the library).
	- `contribute(string bookTitle)`: adds a book to the `bookCollection` with UID of the requested scan.
	- `checkOut()`: checks out book (from UID of next scan)  to `msg.sender`. Initiates a `Rental.sol` contract with the duration and late fees of this rental.
	- `checkIn()`: dual of `checkOut`. Closes the outstanding `Rental.sol` contract.
- `contracts/Rental.sol	`: basic rental agreement between a user and the library. Contains duration and late fee params. 

Contracts were deployed to kovan using remix.

### Frontend

Straightforward web3 interface to the `Library.sol` contract methods. 

## Challenges I ran into
**Running the external adapter on a remote chainlink node**: 

We were first using the services of a node operator in Spain (we're located in PST), which massively slowed down the debug latency for the external adapter <--> smart contract interface. Eventually, we bit the bullet on operating our own chainlink node, which worked very smoothly after initial setup.

**Smart contract design**: 

The root of this problem is that real time (RT) embedded systems (e.g. the RFID scanner) and smart contracts have very different design patterns. As "real time" implies, timing is crucial for clean embedded designs, but smart contracts have well defined execution patterns (i.e. no delay function), which made linking the two modules notably difficult. Eventually, chainlink's external node design helped motivate a callback function style of the RFID system in the contract. 

## Accomplishments that I'm proud of

**Designing a smart contract that interfaces with a real time sensor**: 

We believe that projects of this style really unlock the potential of blockchain as digital infrastructure for analog systems. Though book rentals and RFID scanners are a very specific application of this principle, the smart contract and hardware interface is relevant to everywhere from supply chain to decentralized power systems and more.

**Designing a system that integrated components vertically across the chainlink stack**: 

We custom built nearly all modules for this project from the external adapter/hardware interface to the smart contracts and frontend. We also set up our own chainlink node. **We really came to appreciate chainlink as a piece of "middleware" by working on both sides of its stack**.


## What's next for The Open Library Project

**Tamper proof hardware/general robustness**: 

The obvious flaw in our current design is that the RFID chips can be easily removed, and there is no physical security (something like a lock) on the Little Free Library. This is quite a hard problem to solve, and it motivated the product improvements below.

It's also worth mentioning that there are probably many other ways to game either our hardware or software configurations. Another principle that motivated our redesigns is that there should be as low financial and security overhead, at first. These are the hardest parts to get right, and also get in the way of scaling the network.

**Product improvments**: 

Though RFID proved to be an interesting test bench for the project, it probably unneccasarily overcomplicates the system design and end UX. It also makes the project hard to scale to new Little Free Library locations because of the physical hardware that must be bought and installed. We also think that payments should be removed in the beta version, in favor of scaling the network as easily as possible.

Using QR code stickers on books and a phone app scanner, we can essentially scale for free (minus the small cost of distributing QR stickers). This would dramatically simplify contributing books to the network, and overall make the process much more intuitive.

**Expanding past books**: 

We're also excited about the larger potential of a peer to peer rental platform (as described in the inspiration section). We think that this is a hole in the market that blockchain enabled hardware can uniquely fill the void. This will involve new hardware (e.g. GPS scanners) and much more robust smart contracts