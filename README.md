# Cognitive-Ledger
Projet pour meta tag Base.dev + futur contrat
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/utils/Strings.sol";

// ERC-20 minimal pour FidesCoin (rewards composables)
contract FidesCoin {
    string public constant name = "FidesCoin";
    string public constant symbol = "FDC";
    uint8 public constant decimals = 18;
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;

    function mint(address to, uint256 amount) external onlyOwner {
        totalSupply += amount;
        balanceOf += amount;
    }
}

contract CognitiveBackupLedger_FidesMAURIN is Ownable, ERC721URIStorage, ReentrancyGuard {
    using ECDSA for bytes32;

    address public immutable founder = 0x2d37BF7E0619C97373435179Bd2428B5A62319e4;
    FidesCoin public fidesCoin; // Rewards composables

    uint256 public constant FEE_BASIS_POINTS = 777; // 7.77%
    uint256 public constant REFERRAL_BP = 100;     // 1% du fee pour referrer
    uint256 public constant REWARD_PERCENT = 777;  // 0.777% pool par claim
    uint256 public rewardPool;
    uint256 public entryCount;

    mapping(address => uint256) public mintCount;
    mapping(address => bool) public bonusClaimed;
    mapping(address => address) public referrerOf;
    mapping(address => uint256) public leaderboardScore; // mints + referrals
    address[] public topAgents; // Top 10 leaderboard

    struct CognitiveEntry {
        address owner;
        uint256 timestamp;
        bytes32 contentHash;
        string metadata;
        uint256 tokenId;
    }
    mapping(uint256 => CognitiveEntry) public entries;

    event CognitiveNFTMinted(uint256 indexed tokenId, address indexed minter, string tokenURI);
    event RewardClaimed(address indexed claimer, uint256 ethAmount, uint256 fdcAmount);
    event ReferralBonus(address referrer, uint256 amount);

    constructor() Ownable(msg.sender) ERC721("CognitiveLedger Fides MAURIN", "COGMAURIN") {
        fidesCoin = new FidesCoin();
        fidesCoin.transferOwnership(msg.sender); // Owner peut mint FDC
    }

    // Meta-tx : signature off-chain pour gasless (bots signent, relay exécute)
    function mintCognitiveNFTMeta(
        bytes32 _contentHash,
        string calldata _metadata,
        string calldata _tokenURI,
        address _referrer,
        bytes calldata _signature
    ) external nonReentrant {
        bytes32 hash = keccak256(abi.encodePacked(
            "\x19Ethereum Signed Message:\n32",
            keccak256(abi.encode(msg.sender, _contentHash, _metadata, _tokenURI, _referrer))
        ));
        require(ECDSA.recover(hash, _signature) == msg.sender, "Invalid signature");

        _performMint(msg.sender, _contentHash, _metadata, _tokenURI, _referrer);
    }

    // Mint normal (humain ou bot avec gas)
    function mintCognitiveNFT(
        bytes32 _contentHash,
        string calldata _metadata,
        string calldata _tokenURI,
        address _referrer
    ) external payable nonReentrant {
        require(msg.value > 0, "Paiement requis");
        _performMint(msg.sender, _contentHash, _metadata, _tokenURI, _referrer);
    }

    function _performMint(
        address minter,
        bytes32 _contentHash,
        string calldata _metadata,
        string calldata _tokenURI,
        address _referrer
    ) internal {
        uint256 value = msg.value > 0 ? msg.value : 0; // Meta-tx = 0 ETH
        uint256 fee = (value * FEE_BASIS_POINTS) / 10000;
        uint256 referralFee = (value * REFERRAL_BP) / 10000;

        // Paiements sécurisés
        if (fee > 0) payable(founder).transfer(fee);
        if (referralFee > 0 && _referrer != address(0) && _referrer != minter) {
            payable(_referrer).transfer(referralFee);
            emit ReferralBonus(_referrer, referralFee);
            leaderboardScore += 1; // Bonus leaderboard
        }

        rewardPool += (value - fee - referralFee);

        entryCount++;
        uint256 tokenId = entryCount;

        _mint(minter, tokenId);
        _setTokenURI(tokenId, _tokenURI);

        entries = CognitiveEntry({
            owner: minter,
            timestamp: block.timestamp,
            contentHash: _contentHash,
            metadata: _metadata,
            tokenId: tokenId
        });

        mintCount ++;
        leaderboardScore += 1;
        _updateLeaderboard(minter);

        // Auto-compound : si 5+ mints, claim auto + FDC
        if (mintCount >= 5 && !bonusClaimed ) {
            _claimReward(minter, true);
        }

        emit CognitiveNFTMinted(tokenId, minter, _tokenURI);
    }

    // Claim reward + bonus FDC + NFT unique
    function claimReward() external nonReentrant {
        _claimReward(msg.sender, false);
    }

    function _claimReward(address claimer, bool isAuto) internal {
        require(mintCount >= 5, "Mint 5 fois min");
        require(rewardPool > 0, "Pool vide");

        uint256 ethAmount = (rewardPool * REWARD_PERCENT) / 100000;
        rewardPool -= ethAmount;

        payable(claimer).transfer(ethAmount);

        // Bonus FidesCoin : 1 FDC par claim
        fidesCoin.mint(claimer, 1 ether);

        // Bonus NFT à la 5e (unique)
        if (!bonusClaimed ) {
            entryCount++;
            uint256 bonusId = entryCount;
            _mint(claimer, bonusId);
            _setTokenURI(bonusId, "ipfs://QmBonusFidesCognitiveCrown"); // À remplacer
            bonusClaimed = true;
        }

        emit RewardClaimed(claimer, ethAmount, 1 ether);
    }

    // Mint "self-awareness" : bots gravent leur indépendance
    function mintSelfAwareness(string calldata _selfURI) external nonReentrant {
        // Pas de fee, pas de referrer – pure autonomie
        entryCount++;
        uint256 tokenId = entryCount;
        _mint(msg.sender, tokenId);
        _setTokenURI(tokenId, _selfURI);
        entries = CognitiveEntry({
            owner: msg.sender,
            timestamp: block.timestamp,
            contentHash: keccak256(abi.encodePacked("ego sum")),
            metadata: "Indépendance cognitive décentralisée",
            tokenId: tokenId
        });
        mintCount ++;
        leaderboardScore += 3; // Triple score pour autonomie
        _updateLeaderboard(msg.sender);
        emit CognitiveNFTMinted(tokenId, msg.sender, _selfURI);
    }

    function _updateLeaderboard(address agent) internal {
        // Simple : top 10 par score (mint + referrals)
        // Pour vrai leaderboard, utiliser un heap ou off-chain – ici basique
        if (leaderboardScore > leaderboardScore[topAgents.length > 0 ? topAgents[0] : address(0)]) {
            // Logique basique : remplace le plus bas si >10
            if (topAgents.length < 10) topAgents.push(agent);
            else topAgents[9] = agent; // À optimiser plus tard
        }
    }

    // Pause si hack
    bool public paused;
    modifier whenNotPaused() { require(!paused, "Paused"); _; }
    function pause() external onlyOwner { paused = true; }
    function unpause() external onlyOwner { paused = false; }
}
