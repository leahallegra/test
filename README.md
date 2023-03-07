# test
Testing audit tools
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/math/SafeMath.sol";

contract ChamePresale is Ownable {
    using SafeERC20 for IERC20;
    using SafeMath for uint256;

    uint256 public immutable MIN_USD;
    uint256 public constant NO_AML_MAX = 10_000;
    uint256 public constant CHAME_AMOUNT = 6_000_000 * 10 ** 9;
    uint256 public constant COLLECT_DURATION = 7 days;

    mapping(address => uint256) private balances;
    mapping(address => bool) private claimed;

    bool collectStarted = false;
    bool collectEnded = false;
    bool failed = false;

    uint256 public startTime;
    address public immutable usdAddress;
    address public immutable chame;
    address public receiver;

    uint256 private totalCollectedAtEnd;

    constructor(
        address _usdAddress,
        address _receiver,
        uint8 _usdDecimals,
        address _chame
    ) {
        MIN_USD = 200_000 * 10 ** _usdDecimals;
        usdAddress = _usdAddress;
        receiver = _receiver;
        chame = _chame;
    }

    function setReceiver(address _receiver) external onlyOwner {
        receiver = _receiver;
    }

    //check balance paid in - will be needed for token distribution
    function blanceOf(address user) external view returns (uint256) {
        return balances[user];
    }

    function userClaimed(address user) external view returns (bool) {
        return claimed[user];
    }

    function userTokens(address user) public view returns (uint256) {
        if (balances[user] == 0) {
            return 0;
        }
        return CHAME_AMOUNT.mul(balances[user]).div(totalCollected());
    }

    //total collected by this contract and KYC/AML-ed collect to owner address
    function totalCollected() public view returns (uint256) {
        if (totalCollectedAtEnd > 0) {
            return totalCollectedAtEnd;
        }
        return
            IERC20(usdAddress).balanceOf(address(this)) +
            IERC20(usdAddress).balanceOf(owner());
    }

    function start() external onlyOwner {
        require(!collectStarted, "Collect started");
        collectStarted = true;
        startTime = block.timestamp;

        IERC20(chame).transferFrom(_msgSender(), address(this), CHAME_AMOUNT);
        // if fee applies, then revert
        require(
            IERC20(chame).balanceOf(address(this)) == CHAME_AMOUNT,
            "Incorrect chame amount"
        );
    }

    function collectTimePassed() public view returns (bool) {
        return block.timestamp >= startTime + COLLECT_DURATION;
    }

    function collect(uint256 amount) external {
        require(collectStarted, "Collect not started");
        require(!collectTimePassed(), "Collect ended");
        //if you want pay in more than NO_AML_MAX - contact staff to KYC/AML
        //and pay directly to owner address
        //not KYC/AML-ed payments will be treated as a donation
        require(amount <= NO_AML_MAX, "Need KYC/AML");

        uint256 before = IERC20(usdAddress).balanceOf(address(this));
        IERC20(usdAddress).transferFrom(_msgSender(), address(this), amount);
        uint256 finalAmount = IERC20(usdAddress).balanceOf(address(this)).sub(
            before
        );
        balances[_msgSender()] += finalAmount;

        require(balances[_msgSender()] <= NO_AML_MAX, "Need KYC/AML");
    }

    function claim() external {
        require(!failed, "Collect failed");
        require(collectEnded, "Collect not ended");
        require(balances[_msgSender()] > 0, "Did not participate");
        require(!claimed[_msgSender()], "Already claimed");

        uint256 toClaim = userTokens(_msgSender());
        claimed[_msgSender()] = true;
        IERC20(chame).transfer(_msgSender(), toClaim);
    }

    //withdraw USD if collection failed
    function withdraw() external {
        require(failed, "Collect not failed");
        uint256 amount = balances[_msgSender()];
        balances[_msgSender()] = 0;
        IERC20(usdAddress).transfer(_msgSender(), amount);
    }

    //end collecting - take USD or fail and allow to withdraw
    function end() external onlyOwner {
        require(collectStarted, "Collect not started");
        require(!collectEnded, "Collect ended");
        require(collectTimePassed(), "Time not passed");
        collectEnded = true;
        totalCollectedAtEnd = totalCollected();
        if (totalCollectedAtEnd < MIN_USD) {
            failed = true;
            IERC20(chame).transfer(
                receiver,
                IERC20(chame).balanceOf(address(this))
            );
        } else {
            IERC20(usdAddress).transfer(
                receiver,
                IERC20(usdAddress).balanceOf(address(this))
            );
        }
    }
}
