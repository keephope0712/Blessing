# Blessing
Blessing.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract BaseBlessingBeggar {
    address public owner;
    
    uint256 public totalDonations;
    uint256 public totalDonors;
    uint256 public blessingCount;

    mapping(address => uint256) public donorTotal;
    mapping(address => uint256) public donorBlessingCount;
    mapping(address => uint256) public lastDonationTime;
    
    mapping(uint256 => Blessing) public blessings;
    
    struct Blessing {
        address donor;
        uint256 amount;
        uint256 timestamp;
        string message;
        uint8 tier;
    }

    event Donated(address indexed donor, uint256 amount, uint8 tier, string message);
    event FundsWithdrawn(address to, uint256 amount);

    constructor() {
        owner = msg.sender;
    }

    function giveBlessing(string calldata _customMessage) external payable {
        require(msg.value > 0, "Donation must be > 0 ETH");

        totalDonations += msg.value;
        if (donorTotal[msg.sender] == 0) {
            totalDonors++;
        }

        donorTotal[msg.sender] += msg.value;
        donorBlessingCount[msg.sender]++;
        lastDonationTime[msg.sender] = block.timestamp;

        uint8 tier = _calculateTier(msg.value);
        string memory finalMessage = _generateBlessing(tier, _customMessage);

        blessingCount++;
        blessings[blessingCount] = Blessing({
            donor: msg.sender,
            amount: msg.value,
            timestamp: block.timestamp,
            message: finalMessage,
            tier: tier
        });

        emit Donated(msg.sender, msg.value, tier, finalMessage);
    }

    function _calculateTier(uint256 amount) internal pure returns (uint8) {
        if (amount >= 0.1 ether) return 5;
        if (amount >= 0.01 ether) return 4;
        if (amount >= 0.001 ether) return 3;
        if (amount >= 0.0001 ether) return 2;
        return 1;
    }

    function _generateBlessing(uint8 tier, string memory customMsg) internal pure returns (string memory) {
        if (bytes(customMsg).length > 0) {
            return customMsg;
        }

        if (tier == 5) return "Legendary fortune! The universe bows before your generosity.";
        if (tier == 4) return "Divine blessings descend upon you. Wealth, health, and joy multiply.";
        if (tier == 3) return "Great luck flows your way. Opportunities knock loudly today.";
        if (tier == 2) return "Good energy surrounds you. Smile - today is blessed.";
        return "A kind blessing of gratitude. May your path be filled with light.";
    }

    function getLatestBlessings(uint256 limit) external view returns (Blessing[] memory) {
        if (blessingCount == 0) {
            return new Blessing[](0);
        }
        
        uint256 start = blessingCount > limit ? blessingCount - limit + 1 : 1;
        uint256 size = blessingCount - start + 1;
        Blessing[] memory list = new Blessing[](size);
        
        for (uint256 i = 0; i < size; i++) {
            list[i] = blessings[start + i];
        }
        return list;
    }

    function getMyStats(address _donor) external view returns (
        uint256 totalDonated,
        uint256 blessingCount_,
        uint8 currentTier,
        uint256 timeSinceLastDonation
    ) {
        totalDonated = donorTotal[_donor];
        blessingCount_ = donorBlessingCount[_donor];
        currentTier = donorTotal[_donor] >= 0.1 ether ? 5 : 
                     donorTotal[_donor] >= 0.01 ether ? 4 : 3;
        timeSinceLastDonation = block.timestamp - lastDonationTime[_donor];
    }

    function getContractBalance() external view returns (uint256) {
        return address(this).balance;
    }

    function withdraw(uint256 _amount) external onlyOwner {
        require(_amount <= address(this).balance, "Insufficient balance");
        _safeTransfer(owner, _amount);
        emit FundsWithdrawn(owner, _amount);
    }

    function withdrawAll() external onlyOwner {
        uint256 bal = address(this).balance;
        _safeTransfer(owner, bal);
        emit FundsWithdrawn(owner, bal);
    }

    function _safeTransfer(address to, uint256 amount) internal {
        (bool success, ) = payable(to).call{value: amount}("");
        require(success, "ETH transfer failed");
    }

    function transferOwnership(address newOwner) external onlyOwner {
        require(newOwner != address(0), "Zero address");
        owner = newOwner;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    receive() external payable {
        if (msg.value > 0) {
            _processDirectDonation();
        }
    }

    function _processDirectDonation() internal {
        totalDonations += msg.value;
        if (donorTotal[msg.sender] == 0) {
            totalDonors++;
        }

        donorTotal[msg.sender] += msg.value;
        donorBlessingCount[msg.sender]++;
        lastDonationTime[msg.sender] = block.timestamp;

        uint8 tier = _calculateTier(msg.value);
        
        string memory emptyMsg = "";
        string memory finalMessage = _generateBlessing(tier, emptyMsg);

        blessingCount++;
        blessings[blessingCount] = Blessing({
            donor: msg.sender,
            amount: msg.value,
            timestamp: block.timestamp,
            message: finalMessage,
            tier: tier
        });

        emit Donated(msg.sender, msg.value, tier, finalMessage);
    }
}
cannot encode empty arguments
