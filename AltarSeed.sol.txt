// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/*
 Simple, verified-friendly ERC20 with Ownable, Mint, Burn, Pause.
 Designed for easy verification on explorers (flattening/pasting single file).
*/

interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}

/// @notice Basic Context
abstract contract Context {
    function _msgSender() internal view virtual returns (address) {
        return msg.sender;
    }
}

/// @notice Ownable (simple)
contract Ownable is Context {
    address private _owner;

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    constructor() {
        _transferOwnership(_msgSender());
    }

    function owner() public view returns (address) {
        return _owner;
    }

    modifier onlyOwner() {
        require(owner() == _msgSender(), "Ownable: caller is not the owner");
        _;
    }

    function renounceOwnership() public onlyOwner {
        _transferOwnership(address(0));
    }

    function transferOwnership(address newOwner) public onlyOwner {
        require(newOwner != address(0), "Ownable: new owner is the zero address");
        _transferOwnership(newOwner);
    }

    function _transferOwnership(address newOwner) internal {
        address old = _owner;
        _owner = newOwner;
        emit OwnershipTransferred(old, newOwner);
    }
}

/// @notice Pausable
contract Pausable is Context {
    event Paused(address account);
    event Unpaused(address account);

    bool private _paused;

    constructor() { _paused = false; }

    function paused() public view returns (bool) { return _paused; }

    modifier whenNotPaused() {
        require(!_paused, "Pausable: paused");
        _;
    }

    modifier whenPaused() {
        require(_paused, "Pausable: not paused");
        _;
    }

    function _pause() internal whenNotPaused {
        _paused = true;
        emit Paused(_msgSender());
    }

    function _unpause() internal whenPaused {
        _paused = false;
        emit Unpaused(_msgSender());
    }
}

/// @notice Minimal ERC20 implementation (simple, verifiable)
contract ERC20 is Context, IERC20 {
    mapping(address => uint256) public override balanceOf;
    mapping(address => mapping(address => uint256)) public override allowance;
    uint256 public override totalSupply;

    string public name;
    string public symbol;
    uint8 public decimals = 18;

    constructor(string memory _name, string memory _symbol) {
        name = _name;
        symbol = _symbol;
    }

    function transfer(address to, uint256 amount) public virtual override returns (bool) {
        _transfer(_msgSender(), to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) public virtual override returns (bool) {
        allowance[_msgSender()][spender] = amount;
        emit Approval(_msgSender(), spender, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) public virtual override returns (bool) {
        uint256 allowed = allowance[from][_msgSender()];
        if (allowed != type(uint256).max) {
            require(allowed >= amount, "ERC20: transfer amount exceeds allowance");
            allowance[from][_msgSender()] = allowed - amount;
        }
        _transfer(from, to, amount);
        return true;
    }

    // internal
    function _transfer(address from, address to, uint256 amount) internal virtual {
        require(from != address(0), "ERC20: transfer from zero");
        require(to != address(0), "ERC20: transfer to zero");
        uint256 fromBal = balanceOf[from];
        require(fromBal >= amount, "ERC20: transfer amount exceeds balance");
        unchecked {
            balanceOf[from] = fromBal - amount;
            balanceOf[to] += amount;
        }
        emit Transfer(from, to, amount);
    }

    function _mint(address to, uint256 amount) internal virtual {
        require(to != address(0), "ERC20: mint to zero");
        totalSupply += amount;
        balanceOf[to] += amount;
        emit Transfer(address(0), to, amount);
    }

    function _burn(address from, uint256 amount) internal virtual {
        require(from != address(0), "ERC20: burn from zero");
        uint256 fromBal = balanceOf[from];
        require(fromBal >= amount, "ERC20: burn amount exceeds balance");
        unchecked {
            balanceOf[from] = fromBal - amount;
            totalSupply -= amount;
        }
        emit Transfer(from, address(0), amount);
    }
}

/// @title ALTAR SEED — simple ERC20 you can verify
contract AltarSeed is ERC20, Ownable, Pausable {
    event Mint(address indexed to, uint256 amount);
    event Burned(address indexed from, uint256 amount);
    event PausedBy(address indexed admin);
    event UnpausedBy(address indexed admin);

    constructor(uint256 initialSupply) ERC20("ALTAR SEED", "ALTAR") {
        // initialSupply should be provided in wei (i.e., tokens * 10**18)
        _mint(_msgSender(), initialSupply);
    }

    // owner-only mint
    function mint(address to, uint256 amount) external onlyOwner whenNotPaused {
        _mint(to, amount);
        emit Mint(to, amount);
    }

    // burn from caller
    function burn(uint256 amount) external whenNotPaused {
        _burn(_msgSender(), amount);
        emit Burned(_msgSender(), amount);
    }

    // owner pause/unpause
    function pause() external onlyOwner {
        _pause();
        emit PausedBy(_msgSender());
    }

    function unpause() external onlyOwner {
        _unpause();
        emit UnpausedBy(_msgSender());
    }

    // override transfer to respect pause
    function transfer(address to, uint256 amount) public virtual override whenNotPaused returns (bool) {
        return super.transfer(to, amount);
    }

    function transferFrom(address from, address to, uint256 amount) public virtual override whenNotPaused returns (bool) {
        return super.transferFrom(from, to, amount);
    }

    // convenience: composer function to get human-friendly supply
    function human(uint256 n) public pure returns (uint256) {
        return n * (10 ** 18);
    }
}
