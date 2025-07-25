# 072025

# MLMStoreFactory

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MLMStoreFactory is Ownable, ReentrancyGuard {
    uint256 private _storeCounter;
    uint256 public factoryCommissionRate = 5;
    address[] public allStores;
    mapping(address => bool) public isStore;
    mapping(address => uint256) public storeCommissions;
    mapping(address => address) public storeCreators;
    
    uint256 public storeCreationPrice;
    bool public storeCreationEnabled;
    uint256 public totalCreationFees;
    bool public globalPause;

    event StoreCreated(address indexed storeAddress, address indexed creator, uint256 storeId);
    event FactoryCommissionUpdated(uint256 newRate);
    event CommissionWithdrawn(uint256 amount);
    event CommissionReceived(address indexed store, uint256 amount);
    event StoreCreationPriceUpdated(uint256 newPrice);
    event StoreCreationEnabledUpdated(bool enabled);
    event GlobalPauseUpdated(bool paused);

    constructor() Ownable(msg.sender) {
        storeCreationEnabled = false;
    }

    function setFactoryCommissionRate(uint256 _newRate) external onlyOwner {
        require(_newRate <= 20, "COMMISSION_TOO_HIGH");
        factoryCommissionRate = _newRate;
        emit FactoryCommissionUpdated(_newRate);
    }

    function setStoreCreationPrice(uint256 _price) external onlyOwner {
        storeCreationPrice = _price;
        emit StoreCreationPriceUpdated(_price);
    }

    function setStoreCreationEnabled(bool _enabled) external onlyOwner {
        storeCreationEnabled = _enabled;
        emit StoreCreationEnabledUpdated(_enabled);
    }

    function setGlobalPause(bool _paused) external onlyOwner {
        globalPause = _paused;
        emit GlobalPauseUpdated(_paused);
    }

    function createStore(
        address[] memory _owners,
        uint256 _requiredSignatures,
        address _treasury,
        string memory _storeName
    ) external payable returns (address) {
        require(!globalPause, "GLOBAL_PAUSE_ACTIVE");
        require(_owners.length > 0, "NO_OWNERS");
        require(_requiredSignatures > 0 && _requiredSignatures <= _owners.length, "INVALID_SIGNATURES");
        require(_treasury != address(0), "INVALID_TREASURY");
        
        if (storeCreationEnabled) {
            require(msg.value >= storeCreationPrice, "INSUFFICIENT_PAYMENT");
            totalCreationFees += storeCreationPrice;
            
            if (msg.value > storeCreationPrice) {
                uint256 excess = msg.value - storeCreationPrice;
                (bool success, ) = payable(msg.sender).call{value: excess}("");
                require(success, "REFUND_FAILED");
            }
        } else {
            require(msg.value == 0, "PAYMENT_NOT_REQUIRED");
        }

        _storeCounter++;
        uint256 storeId = _storeCounter;
        MLMStore newStore = new MLMStore(
            _owners,
            _requiredSignatures,
            _treasury,
            address(this),
            factoryCommissionRate,
            _storeName,
            storeId
        );
        
        allStores.push(address(newStore));
        isStore[address(newStore)] = true;
        storeCreators[address(newStore)] = msg.sender;
        
        emit StoreCreated(address(newStore), msg.sender, storeId);
        return address(newStore);
    }

    function collectCommission() external payable nonReentrant {
        require(!globalPause, "GLOBAL_PAUSE_ACTIVE");
        require(isStore[msg.sender], "NOT_STORE");
        require(msg.value > 0, "NO_COMMISSION");
        storeCommissions[msg.sender] += msg.value;
        emit CommissionReceived(msg.sender, msg.value);
    }

    function withdrawCommissions() external onlyOwner nonReentrant {
        uint256 amount = address(this).balance;
        require(amount > 0, "NO_COMMISSIONS");
        (bool success, ) = payable(owner()).call{value: amount}("");
        require(success, "TRANSFER_FAILED");
        emit CommissionWithdrawn(amount);
    }

    function getStoreCount() external view returns (uint256) {
        return allStores.length;
    }

    function getStoreInfo(address _store) external view returns (
        bool isValid,
        address creator,
        uint256 commissions
    ) {
        return (isStore[_store], storeCreators[_store], storeCommissions[_store]);
    }

    receive() external payable {}
}

contract MLMStore is ReentrancyGuard {
    enum ProductType { DIGITAL, PHYSICAL, SUBSCRIPTION }
    
    struct Product {
        uint256 id;
        uint256 price;
        ProductType pType;
        uint256 duration;
        string metadataIPFS;
        bool isActive;
        uint256 created;
    }
    
    struct Subscription {
        address user;
        uint256 productId;
        uint256 startDate;
        uint256 endDate;
    }
    
    struct Buyer {
        address referrer;
        uint256 totalSpent;
        uint256 totalReferrals;
        uint256 totalCommissions;
        bool isActive;
    }
    
    struct MultisigTransaction {
        address to;
        uint256 value;
        bytes data;
        bool executed;
        uint256 confirmations;
        mapping(address => bool) isConfirmed;
        string description;
    }

    uint256 private _productCounter;
    uint256 private _transactionCounter;
    string public storeName;
    uint256 public storeId;
    address payable public factory;
    address public treasury;
    uint256 public factoryCommissionRate;
    
    address[] public owners;
    mapping(address => bool) public isOwner;
    uint256 public requiredSignatures;
    mapping(uint256 => MultisigTransaction) public transactions;
    
    mapping(uint256 => Product) public products;
    mapping(address => Buyer) public buyers;
    mapping(uint256 => Subscription) public subscriptions;
    mapping(address => uint256[]) public userSubscriptions;
    
    uint256 public referralCommissionRate = 10;
    uint256 public constant MAX_REFERRAL_LEVELS = 5;
    uint256[] public levelCommissions = [50, 20, 15, 10, 5];
    
    bool public emergencyPause;

    event ProductCreated(uint256 indexed productId, uint256 price, ProductType pType);
    event ProductPurchased(uint256 indexed productId, address indexed buyer, address indexed referrer, uint256 amount);
    event SubscriptionCreated(uint256 indexed subscriptionId, address indexed user, uint256 productId);
    event ReferralCommissionPaid(address indexed referrer, address indexed buyer, uint256 amount, uint256 level);
    event MultisigTransactionSubmitted(uint256 indexed transactionId, address indexed submitter);
    event MultisigTransactionConfirmed(uint256 indexed transactionId, address indexed owner);
    event MultisigTransactionExecuted(uint256 indexed transactionId);
    event OwnerAdded(address indexed newOwner);
    event OwnerRemoved(address indexed removedOwner);
    event RequiredSignaturesChanged(uint256 newRequired);
    event ProductStatusChanged(uint256 indexed productId, bool active);
    event LevelCommissionsUpdated();
    event EmergencyPauseSet(bool paused);

    modifier onlyOwner() {
        require(isOwner[msg.sender], "NOT_OWNER");
        _;
    }
    
    modifier transactionExists(uint256 _transactionId) {
        require(_transactionId > 0 && _transactionId <= _transactionCounter, "TRANSACTION_NOT_EXISTS");
        _;
    }
    
    modifier notExecuted(uint256 _transactionId) {
        require(!transactions[_transactionId].executed, "TRANSACTION_EXECUTED");
        _;
    }
    
    modifier notConfirmed(uint256 _transactionId) {
        require(!transactions[_transactionId].isConfirmed[msg.sender], "TRANSACTION_CONFIRMED");
        _;
    }

    modifier whenNotPaused() {
        require(!emergencyPause, "STORE_PAUSED");
        _;
    }

    constructor(
        address[] memory _owners,
        uint256 _requiredSignatures,
        address _treasury,
        address _factory,
        uint256 _factoryCommissionRate,
        string memory _storeName,
        uint256 _storeId
    ) {
        require(_owners.length > 0, "NO_OWNERS");
        require(_requiredSignatures > 0 && _requiredSignatures <= _owners.length, "INVALID_SIGNATURES");
        require(_treasury != address(0), "INVALID_TREASURY");
        require(_factory != address(0), "INVALID_FACTORY");
        
        for (uint256 i = 0; i < _owners.length; i++) {
            require(_owners[i] != address(0), "INVALID_OWNER");
            require(!isOwner[_owners[i]], "DUPLICATE_OWNER");
            isOwner[_owners[i]] = true;
            owners.push(_owners[i]);
        }
        
        requiredSignatures = _requiredSignatures;
        treasury = _treasury;
        factory = payable(_factory);
        factoryCommissionRate = _factoryCommissionRate;
        storeName = _storeName;
        storeId = _storeId;
    }

    function setEmergencyPause(bool _paused) external onlyOwner {
        emergencyPause = _paused;
        emit EmergencyPauseSet(_paused);
    }

    function setLevelCommissions(uint256[] calldata _newLevels) external onlyOwner {
        require(_newLevels.length <= MAX_REFERRAL_LEVELS, "TOO_MANY_LEVELS");
        uint256 total;
        for (uint i = 0; i < _newLevels.length; i++) {
            total += _newLevels[i];
        }
        require(total <= 100, "TOTAL_EXCEEDS_100");
        levelCommissions = _newLevels;
        emit LevelCommissionsUpdated();
    }

    function toggleProductActive(uint256 _productId) external onlyOwner {
        require(_productId > 0 && _productId <= _productCounter, "INVALID_PRODUCT");
        products[_productId].isActive = !products[_productId].isActive;
        emit ProductStatusChanged(_productId, products[_productId].isActive);
    }

    function createProduct(
        uint256 _price,
        ProductType _pType,
        uint256 _duration,
        string memory _metadataIPFS
    ) external onlyOwner returns (uint256) {
        require(_price > 0, "INVALID_PRICE");
        require(bytes(_metadataIPFS).length > 0, "INVALID_IPFS_HASH");
        
        _productCounter++;
        uint256 productId = _productCounter;
        
        products[productId] = Product({
            id: productId,
            price: _price,
            pType: _pType,
            duration: _duration,
            metadataIPFS: _metadataIPFS,
            isActive: true,
            created: block.timestamp
        });
        
        emit ProductCreated(productId, _price, _pType);
        return productId;
    }

    function purchaseProduct(uint256 _productId, address _referrer) 
        external 
        payable 
        nonReentrant 
        whenNotPaused 
    {
        require(!MLMStoreFactory(factory).globalPause(), "GLOBAL_PAUSE_ACTIVE");
        
        Product storage product = products[_productId];
        require(product.id > 0, "PRODUCT_NOT_EXIST");
        require(product.isActive, "PRODUCT_NOT_ACTIVE");
        require(msg.value >= product.price, "INSUFFICIENT_PAYMENT");
        
        if (buyers[msg.sender].totalSpent == 0) {
            buyers[msg.sender] = Buyer({
                referrer: _referrer,
                totalSpent: 0,
                totalReferrals: 0,
                totalCommissions: 0,
                isActive: true
            });
        }
        
        buyers[msg.sender].totalSpent += product.price;
        
        _processReferralCommissions(msg.sender, _referrer, product.price);
        
        uint256 factoryCommission = (product.price * factoryCommissionRate) / 100;
        if (factoryCommission > 0) {
            (bool success, ) = factory.call{value: factoryCommission}("");
            if (!success) {
                (bool fallbackSuccess, ) = treasury.call{value: factoryCommission}("");
                require(fallbackSuccess, "FACTORY_COMMISSION_FAILBACK_FAILED");
            }
        }
        
        if (product.pType == ProductType.SUBSCRIPTION) {
            _createSubscription(msg.sender, _productId, product.duration);
        }
        
        uint256 totalCommissions = _getTotalCommissions(product.price);
        uint256 remaining = product.price - totalCommissions - factoryCommission;
        
        if (remaining > 0) {
            (bool treasurySuccess, ) = treasury.call{value: remaining}("");
            require(treasurySuccess, "TREASURY_TRANSFER_FAILED");
        }
        
        if (msg.value > product.price) {
            uint256 refundAmount = msg.value - product.price;
            (bool refundSuccess, ) = msg.sender.call{value: refundAmount}("");
            require(refundSuccess, "REFUND_FAILED");
        }
        
        emit ProductPurchased(_productId, msg.sender, _referrer, product.price);
    }

    function submitTransaction(
        address _to,
        uint256 _value,
        bytes memory _data,
        string memory _description
    ) external onlyOwner returns (uint256) {
        _transactionCounter++;
        uint256 transactionId = _transactionCounter;
        
        MultisigTransaction storage newTransaction = transactions[transactionId];
        newTransaction.to = _to;
        newTransaction.value = _value;
        newTransaction.data = _data;
        newTransaction.executed = false;
        newTransaction.confirmations = 0;
        newTransaction.description = _description;
        
        emit MultisigTransactionSubmitted(transactionId, msg.sender);
        return transactionId;
    }

    function confirmTransaction(uint256 _transactionId) 
        external 
        onlyOwner 
        transactionExists(_transactionId) 
        notConfirmed(_transactionId) 
        notExecuted(_transactionId) 
    {
        transactions[_transactionId].isConfirmed[msg.sender] = true;
        transactions[_transactionId].confirmations++;
        
        emit MultisigTransactionConfirmed(_transactionId, msg.sender);
        
        if (transactions[_transactionId].confirmations >= requiredSignatures) {
            executeTransaction(_transactionId);
        }
    }

    function executeTransaction(uint256 _transactionId) 
        public 
        transactionExists(_transactionId) 
        notExecuted(_transactionId) 
    {
        require(
            transactions[_transactionId].confirmations >= requiredSignatures, 
            "NOT_ENOUGH_CONFIRMATIONS"
        );
        
        transactions[_transactionId].executed = true;
        (bool success, ) = transactions[_transactionId].to.call{value: transactions[_transactionId].value}(
            transactions[_transactionId].data
        );
        require(success, "TRANSACTION_FAILED");
        
        emit MultisigTransactionExecuted(_transactionId);
    }

    function addOwner(address _newOwner) external {
        require(msg.sender == address(this), "ONLY_MULTISIG");
        require(_newOwner != address(0), "INVALID_OWNER");
        require(!isOwner[_newOwner], "ALREADY_OWNER");
        
        isOwner[_newOwner] = true;
        owners.push(_newOwner);
        
        emit OwnerAdded(_newOwner);
    }

    function removeOwner(address _owner) external {
        require(msg.sender == address(this), "ONLY_MULTISIG");
        require(isOwner[_owner], "NOT_OWNER");
        require(owners.length > 1, "CANNOT_REMOVE_LAST_OWNER");
        
        isOwner[_owner] = false;
        for (uint256 i = 0; i < owners.length; i++) {
            if (owners[i] == _owner) {
                owners[i] = owners[owners.length - 1];
                owners.pop();
                break;
            }
        }
        
        if (requiredSignatures > owners.length) {
            requiredSignatures = owners.length;
        }
        
        emit OwnerRemoved(_owner);
    }

    function changeRequiredSignatures(uint256 _newRequired) external {
        require(msg.sender == address(this), "ONLY_MULTISIG");
        require(_newRequired > 0 && _newRequired <= owners.length, "INVALID_SIGNATURES");
        
        requiredSignatures = _newRequired;
        emit RequiredSignaturesChanged(_newRequired);
    }

    // Correction 1 : Ajout du modificateur 'view'
    function checkSubscriptionExpiry(uint256 _subscriptionId) external view {
        require(_subscriptionId > 0, "INVALID_SUBSCRIPTION");
        Subscription storage sub = subscriptions[_subscriptionId];
        require(sub.endDate > 0, "SUBSCRIPTION_NOT_EXIST");
        require(block.timestamp >= sub.endDate, "NOT_EXPIRED_YET");
    }

    // Correction 2 : Remplacement pour réduire la taille du contrat
    function getProductCount() external view returns (uint256) {
        return _productCounter;
    }

    function isSubscriptionActive(uint256 _subscriptionId) external view returns (bool) {
        Subscription storage sub = subscriptions[_subscriptionId];
        return block.timestamp <= sub.endDate;
    }

    function _processReferralCommissions(address _buyer, address _referrer, uint256 _amount) internal {
        if (_referrer == address(0) || _referrer == _buyer) return;
        
        address currentReferrer = _referrer;
        uint256 totalCommissionPaid = 0;
        
        for (uint256 level = 0; level < levelCommissions.length && currentReferrer != address(0); level++) {
            if (!buyers[currentReferrer].isActive) break;
            
            uint256 commission = (_amount * levelCommissions[level] * referralCommissionRate) / 10000;
            if (commission > 0) {
                buyers[currentReferrer].totalCommissions += commission;
                totalCommissionPaid += commission;
                
                (bool success, ) = currentReferrer.call{value: commission}("");
                if (success) {
                    emit ReferralCommissionPaid(currentReferrer, _buyer, commission, level + 1);
                } else {
                    (bool fallbackSuccess, ) = treasury.call{value: commission}("");
                    require(fallbackSuccess, "REFERRAL_COMMISSION_FAILBACK_FAILED");
                }
            }
            
            currentReferrer = buyers[currentReferrer].referrer;
        }
    }
    
    function _createSubscription(address _user, uint256 _productId, uint256 _duration) internal {
        uint256 subscriptionId = uint256(keccak256(abi.encodePacked(_user, _productId, block.timestamp)));
        subscriptions[subscriptionId] = Subscription({
            user: _user,
            productId: _productId,
            startDate: block.timestamp,
            endDate: block.timestamp + _duration
        });
        
        userSubscriptions[_user].push(subscriptionId);
        emit SubscriptionCreated(subscriptionId, _user, _productId);
    }
    
    function _getTotalCommissions(uint256 _amount) internal view returns (uint256) {
        uint256 total = 0;
        for (uint256 i = 0; i < levelCommissions.length; i++) {
            total += (_amount * levelCommissions[i] * referralCommissionRate) / 10000;
        }
        return total;
    }

    function getProductDetails(uint256 _productId) external view returns (Product memory) {
        return products[_productId];
    }
    
    function getBuyerDetails(address _buyer) external view returns (Buyer memory) {
        return buyers[_buyer];
    }
    
    function getOwners() external view returns (address[] memory) {
        return owners;
    }
    
    function getTransactionDetails(uint256 _transactionId) external view returns (
        address to,
        uint256 value,
        bytes memory data,
        bool executed,
        uint256 confirmations,
        string memory description
    ) {
        MultisigTransaction storage txn = transactions[_transactionId];
        return (
            txn.to,
            txn.value,
            txn.data,
            txn.executed,
            txn.confirmations,
            txn.description
        );
    }
    
    function isTransactionConfirmed(uint256 _transactionId, address _owner) external view returns (bool) {
        return transactions[_transactionId].isConfirmed[_owner];
    }
    
    function getUserSubscriptions(address _user) external view returns (uint256[] memory) {
        return userSubscriptions[_user];
    }

    receive() external payable {}
}
