// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/**
 * @title Avalanche
 * @dev A smart contract showcasing Avalanche's consensus and subnet capabilities
 * @author Avalanche Development Team
 */
contract Avalanche {
    
    // State variables
    address public owner;
    uint256 public totalStaked;
    uint256 public totalSubnets;
    uint256 public minStakeAmount;
    uint256 public consensusThreshold;
    
    // Structs
    struct Validator {
        address validatorAddress;
        uint256 stakedAmount;
        uint256 delegatedAmount;
        bool isActive;
        uint256 joinTime;
        uint256 subnetId;
    }
    
    struct Subnet {
        uint256 id;
        string name;
        address creator;
        uint256 validatorCount;
        bool isActive;
        uint256 creationTime;
        uint256 minValidators;
    }
    
    struct ConsensusVote {
        bytes32 proposalId;
        address voter;
        bool vote;
        uint256 timestamp;
        uint256 weight;
    }
    
    // Mappings
    mapping(address => Validator) public validators;
    mapping(uint256 => Subnet) public subnets;
    mapping(address => uint256) public stakes;
    mapping(address => uint256) public delegations;
    mapping(bytes32 => ConsensusVote[]) public consensusVotes;
    mapping(bytes32 => bool) public proposalExecuted;
    mapping(address => bool) public isValidator;
    
    // Arrays
    address[] public validatorList;
    uint256[] public subnetList;
    
    // Events
    event ValidatorJoined(address indexed validator, uint256 stakedAmount, uint256 subnetId);
    event ValidatorLeft(address indexed validator, uint256 unstakedAmount);
    event SubnetCreated(uint256 indexed subnetId, string name, address creator);
    event StakeAdded(address indexed staker, uint256 amount);
    event StakeWithdrawn(address indexed staker, uint256 amount);
    event ConsensusVoteCast(bytes32 indexed proposalId, address voter, bool vote, uint256 weight);
    event ProposalExecuted(bytes32 indexed proposalId, bool result);
    event DelegationMade(address indexed delegator, address indexed validator, uint256 amount);
    
    // Modifiers
    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this function");
        _;
    }
    
    modifier onlyValidator() {
        require(isValidator[msg.sender], "Only validators can call this function");
        _;
    }
    
    modifier validStakeAmount() {
        require(msg.value >= minStakeAmount, "Stake amount below minimum");
        _;
    }
    
    /**
     * @dev Constructor - Initialize the Avalanche network contract
     */
    constructor() {
        owner = msg.sender;
        minStakeAmount = 1 ether; // Minimum stake to become validator
        consensusThreshold = 80; // 80% consensus required
        totalSubnets = 0;
    }
    
    /**
     * @dev Core Function 1: Avalanche Consensus Mechanism
     * @param proposalId Unique identifier for the proposal
     * @param vote Boolean vote (true for yes, false for no)
     */
    function castConsensusVote(bytes32 proposalId, bool vote) public onlyValidator {
        require(!proposalExecuted[proposalId], "Proposal already executed");
        
        // Check if validator already voted
        ConsensusVote[] storage votes = consensusVotes[proposalId];
        for (uint i = 0; i < votes.length; i++) {
            require(votes[i].voter != msg.sender, "Validator already voted");
        }
        
        // Calculate vote weight based on stake
        uint256 voteWeight = stakes[msg.sender] + validators[msg.sender].delegatedAmount;
        
        // Record the vote
        consensusVotes[proposalId].push(ConsensusVote({
            proposalId: proposalId,
            voter: msg.sender,
            vote: vote,
            timestamp: block.timestamp,
            weight: voteWeight
        }));
        
        emit ConsensusVoteCast(proposalId, msg.sender, vote, voteWeight);
        
        // Check if consensus is reached
        _checkConsensus(proposalId);
    }
    
    /**
     * @dev Internal function to check if consensus is reached
     * @param proposalId Proposal to check consensus for
     */
    function _checkConsensus(bytes32 proposalId) internal {
        ConsensusVote[] storage votes = consensusVotes[proposalId];
        uint256 totalVoteWeight = 0;
        uint256 yesVoteWeight = 0;
        
        // Calculate total vote weights
        for (uint i = 0; i < votes.length; i++) {
            totalVoteWeight += votes[i].weight;
            if (votes[i].vote) {
                yesVoteWeight += votes[i].weight;
            }
        }
        
        // Check if minimum participation and consensus threshold met
        if (totalVoteWeight >= (totalStaked * 51) / 100) { // 51% participation
            uint256 consensusPercentage = (yesVoteWeight * 100) / totalVoteWeight;
            
            if (consensusPercentage >= consensusThreshold) {
                proposalExecuted[proposalId] = true;
                emit ProposalExecuted(proposalId, true);
            } else if (consensusPercentage <= (100 - consensusThreshold)) {
                proposalExecuted[proposalId] = true;
                emit ProposalExecuted(proposalId, false);
            }
        }
    }
    
    /**
     * @dev Core Function 2: Subnet Creation and Management
     * @param name Name of the subnet
     * @param minValidators Minimum validators required for the subnet
     */
    function createSubnet(string memory name, uint256 minValidators) public payable validStakeAmount {
        require(minValidators > 0, "Minimum validators must be greater than 0");
        require(bytes(name).length > 0, "Subnet name cannot be empty");
        
        totalSubnets++;
        uint256 subnetId = totalSubnets;
        
        // Create new subnet
        subnets[subnetId] = Subnet({
            id: subnetId,
            name: name,
            creator: msg.sender,
            validatorCount: 0,
            isActive: true,
            creationTime: block.timestamp,
            minValidators: minValidators
        });
        
        subnetList.push(subnetId);
        
        // Creator automatically becomes first validator of the subnet
        _joinAsValidator(msg.sender, msg.value, subnetId);
        
        emit SubnetCreated(subnetId, name, msg.sender);
    }
    
    /**
     * @dev Join an existing subnet as validator
     * @param subnetId ID of the subnet to join
     */
    function joinSubnet(uint256 subnetId) public payable validStakeAmount {
        require(subnets[subnetId].isActive, "Subnet is not active");
        require(!isValidator[msg.sender], "Address is already a validator");
        
        _joinAsValidator(msg.sender, msg.value, subnetId);
    }
    
    /**
     * @dev Internal function to handle validator joining
     * @param validator Address of the validator
     * @param stakeAmount Amount being staked
     * @param subnetId Subnet ID to join
     */
    function _joinAsValidator(address validator, uint256 stakeAmount, uint256 subnetId) internal {
        validators[validator] = Validator({
            validatorAddress: validator,
            stakedAmount: stakeAmount,
            delegatedAmount: 0,
            isActive: true,
            joinTime: block.timestamp,
            subnetId: subnetId
        });
        
        stakes[validator] = stakeAmount;
        isValidator[validator] = true;
        validatorList.push(validator);
        
        // Update subnet and total stakes
        subnets[subnetId].validatorCount++;
        totalStaked += stakeAmount;
        
        emit ValidatorJoined(validator, stakeAmount, subnetId);
    }
    
    /**
     * @dev Core Function 3: Proof-of-Stake with Delegation
     * @param validator Address of validator to delegate to
     */
    function delegateStake(address validator) public payable {
        require(isValidator[validator], "Invalid validator address");
        require(validators[validator].isActive, "Validator is not active");
        require(msg.value > 0, "Delegation amount must be greater than 0");
        
        // Add to delegator's total delegations
        delegations[msg.sender] += msg.value;
        
        // Add to validator's delegated amount
        validators[validator].delegatedAmount += msg.value;
        totalStaked += msg.value;
        
        emit DelegationMade(msg.sender, validator, msg.value);
    }
    
    /**
     * @dev Withdraw delegated stake
     * @param validator Validator to withdraw delegation from
     * @param amount Amount to withdraw
     */
    function withdrawDelegation(address validator, uint256 amount) public {
        require(delegations[msg.sender] >= amount, "Insufficient delegated amount");
        require(validators[validator].delegatedAmount >= amount, "Validator doesn't have enough delegated stake");
        
        delegations[msg.sender] -= amount;
        validators[validator].delegatedAmount -= amount;
        totalStaked -= amount;
        
        payable(msg.sender).transfer(amount);
    }
    
    /**
     * @dev Leave validator set and unstake
     */
    function leaveValidatorSet() public onlyValidator {
        Validator storage validator = validators[msg.sender];
        uint256 stakeAmount = validator.stakedAmount;
        uint256 subnetId = validator.subnetId;
        
        // Ensure minimum validators remain in subnet
        require(subnets[subnetId].validatorCount > subnets[subnetId].minValidators, 
                "Cannot leave: would go below minimum validators");
        
        // Update state
        validator.isActive = false;
        isValidator[msg.sender] = false;
        subnets[subnetId].validatorCount--;
        totalStaked -= stakeAmount;
        
        // Return staked amount
        payable(msg.sender).transfer(stakeAmount);
        
        emit ValidatorLeft(msg.sender, stakeAmount);
    }
    
    /**
     * @dev Get subnet information
     * @param subnetId ID of the subnet
     * @return Subnet information
     */
    function getSubnetInfo(uint256 subnetId) public view returns (
        uint256 id,
        string memory name,
        address creator,
        uint256 validatorCount,
        bool isActive,
        uint256 creationTime,
        uint256 minValidators
    ) {
        Subnet storage subnet = subnets[subnetId];
        return (
            subnet.id,
            subnet.name,
            subnet.creator,
            subnet.validatorCount,
            subnet.isActive,
            subnet.creationTime,
            subnet.minValidators
        );
    }
    
    /**
     * @dev Get validator information
     * @param validatorAddr Address of the validator
     * @return Validator information
     */
    function getValidatorInfo(address validatorAddr) public view returns (
        address validatorAddress,
        uint256 stakedAmount,
        uint256 delegatedAmount,
        bool isActive,
        uint256 joinTime,
        uint256 subnetId
    ) {
        Validator storage validator = validators[validatorAddr];
        return (
            validator.validatorAddress,
            validator.stakedAmount,
            validator.delegatedAmount,
            validator.isActive,
            validator.joinTime,
            validator.subnetId
        );
    }
    
    /**
     * @dev Get consensus votes for a proposal
     * @param proposalId ID of the proposal
     * @return Array of votes
     */
    function getConsensusVotes(bytes32 proposalId) public view returns (ConsensusVote[] memory) {
        return consensusVotes[proposalId];
    }
    
    /**
     * @dev Get total number of validators
     * @return Number of validators
     */
    function getValidatorCount() public view returns (uint256) {
        return validatorList.length;
    }
    
    /**
     * @dev Get all subnet IDs
     * @return Array of subnet IDs
     */
    function getAllSubnets() public view returns (uint256[] memory) {
        return subnetList;
    }
    
    /**
     * @dev Update consensus threshold (only owner)
     * @param newThreshold New consensus threshold percentage
     */
    function updateConsensusThreshold(uint256 newThreshold) public onlyOwner {
        require(newThreshold > 50 && newThreshold <= 100, "Invalid consensus threshold");
        consensusThreshold = newThreshold;
    }
    
    /**
     * @dev Update minimum stake amount (only owner)
     * @param newMinStake New minimum stake amount
     */
    function updateMinStakeAmount(uint256 newMinStake) public onlyOwner {
        require(newMinStake > 0, "Minimum stake must be greater than 0");
        minStakeAmount = newMinStake;
    }
    
    /**
     * @dev Emergency pause subnet (only owner)
     * @param subnetId ID of subnet to pause
     */
    function pauseSubnet(uint256 subnetId) public onlyOwner {
        require(subnets[subnetId].id != 0, "Subnet does not exist");
        subnets[subnetId].isActive = false;
    }
    
    /**
     * @dev Get contract balance
     * @return Contract balance in wei
     */
    function getContractBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
