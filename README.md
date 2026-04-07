# Dice-Betting-Game
Dice Betting Game
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/utils/math/Math.sol";

contract BaseGame is Ownable, ReentrancyGuard {

    uint256 public constant MIN_BET = 0.001 ether;
    uint256 public constant MAX_BET = 0.5 ether;
    uint256 public houseEdge = 300; // 3% = 300 / 10000
    uint256 public totalWagered;
    uint256 public totalPayouts;

    struct GameRound {
        address player;
        uint256 betAmount;
        uint256 targetNumber;
        uint256 rollResult;
        bool isFinished;
        uint256 timestamp;
    }

    mapping(uint256 => GameRound) public gameRounds;
    uint256 public roundCounter;

    mapping(address => uint256) public playerWins;
    mapping(address => uint256) public playerLosses;
    mapping(address => uint256) public playerTotalWagered;

    event GamePlayed(
        uint256 indexed roundId,
        address indexed player,
        uint256 betAmount,
        uint256 targetNumber,
        uint256 rollResult,
        bool won,
        uint256 payout
    );

    event HouseEdgeUpdated(uint256 newHouseEdge);
    event WithdrawETH(address to, uint256 amount);

    constructor() Ownable(msg.sender) {}

    function playDice(uint256 targetNumber) external payable nonReentrant {
        require(msg.value >= MIN_BET && msg.value <= MAX_BET, "Bet amount out of range");
        require(targetNumber >= 1 && targetNumber <= 6, "Target must be 1-6");

        roundCounter++;
        uint256 roll = _getRandomNumber() % 6 + 1;

        uint256 payout = 0;
        bool won = roll == targetNumber;

        if (won) {
            uint256 winAmount = (msg.value * (10000 - houseEdge)) / 10000 * 6;
            payout = winAmount;
            require(address(this).balance >= payout, "Insufficient contract balance");
            (bool success, ) = payable(msg.sender).call{value: payout}("");
            require(success, "Payout failed");
            playerWins[msg.sender]++;
        } else {
            playerLosses[msg.sender]++;
        }

        playerTotalWagered[msg.sender] += msg.value;
        totalWagered += msg.value;

        if (won) {
            totalPayouts += payout;
        }

        gameRounds[roundCounter] = GameRound({
            player: msg.sender,
            betAmount: msg.value,
            targetNumber: targetNumber,
            rollResult: roll,
            isFinished: true,
            timestamp: block.timestamp
        });

        emit GamePlayed(roundCounter, msg.sender, msg.value, targetNumber, roll, won, payout);
    }

    function _getRandomNumber() internal view returns (uint256) {
        return uint256(
            keccak256(
                abi.encodePacked(
                    block.prevrandao,
                    block.timestamp,
                    block.number,
                    msg.sender,
                    roundCounter
                )
            )
        );
    }

    function setHouseEdge(uint256 newEdge) external onlyOwner {
        require(newEdge <= 1000, "House edge too high");
        houseEdge = newEdge;
        emit HouseEdgeUpdated(newEdge);
    }

    function withdrawHouseFunds(uint256 amount) external onlyOwner nonReentrant {
        require(amount <= address(this).balance, "Insufficient balance");
        (bool success, ) = payable(owner()).call{value: amount}("");
        require(success, "Withdraw failed");
        emit WithdrawETH(owner(), amount);
    }

    function getContractBalance() external view returns (uint256) {
        return address(this).balance;
    }

    function getPlayerStats(address player) external view returns (
        uint256 wins,
        uint256 losses,
        uint256 totalWageredAmount
    ) {
        return (
            playerWins[player],
            playerLosses[player],
            playerTotalWagered[player]
        );
    }

    function getLastRound() external view returns (GameRound memory) {
        return gameRounds[roundCounter];
    }

    receive() external payable {}
}
