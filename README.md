# WordWall
WordWall.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Strings.sol";

contract WordWall is ERC721, Ownable {
    using Strings for uint256;

    uint256 public constant MAX_MESSAGE_LENGTH = 500;
    uint256 public constant MINT_PRICE = 0.0003 ether;   // 可调整

    uint256 public totalWords;

    struct Word {
        address author;
        string content;
        uint256 timestamp;
        uint256 likes;
    }

    mapping(uint256 => Word) public words;
    mapping(uint256 => mapping(address => bool)) public hasLiked;

    event WordPosted(uint256 indexed tokenId, address author, string content);
    event Liked(uint256 indexed tokenId, address liker);

    constructor() ERC721("WordWall", "WORD") Ownable(msg.sender) {}

    /// @notice Post a message and mint it as NFT
    function postWord(string calldata _content) external payable {
        require(bytes(_content).length > 0 && bytes(_content).length <= MAX_MESSAGE_LENGTH, 
                "Message must be 1-500 bytes");
        require(msg.value >= MINT_PRICE, "Insufficient ETH");

        totalWords++;
        uint256 tokenId = totalWords;

        words[tokenId] = Word({
            author: msg.sender,
            content: _content,
            timestamp: block.timestamp,
            likes: 0
        });

        _safeMint(msg.sender, tokenId);

        emit WordPosted(tokenId, msg.sender, _content);
    }

    /// @notice Like a word (costs nothing)
    function likeWord(uint256 _tokenId) external {
        require(_exists(_tokenId), "Word does not exist");
        require(!hasLiked[_tokenId][msg.sender], "Already liked");

        hasLiked[_tokenId][msg.sender] = true;
        words[_tokenId].likes++;

        emit Liked(_tokenId, msg.sender);
    }

    /// @notice Get word information
    function getWord(uint256 _tokenId) external view returns (Word memory) {
        require(_exists(_tokenId), "Word does not exist");
        return words[_tokenId];
    }

    /// @notice Get latest words (simple pagination)
    function getLatestWords(uint256 limit) external view returns (Word[] memory) {
        uint256 start = totalWords > limit ? totalWords - limit : 0;
        Word[] memory latest = new Word[](totalWords - start);
        
        for (uint256 i = 0; i < latest.length; i++) {
            latest[i] = words[start + i + 1];
        }
        return latest;
    }

    /// @notice Get most liked words (top 10)
    function getTopWords() external view returns (Word[] memory) {
        Word[] memory top = new Word[](totalWords > 10 ? 10 : totalWords);
        
        // Simple top 10 (not sorted for gas saving, can be improved)
        for (uint256 i = 1; i <= (totalWords > 10 ? 10 : totalWords); i++) {
            top[i-1] = words[i];
        }
        return top;
    }

    /// @notice Token URI - simple on-chain metadata
    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        require(_exists(tokenId), "Token does not exist");
        
        Word memory w = words[tokenId];

        return string(abi.encodePacked(
            "data:application/json;utf8,",
            '{"name":"Word #', tokenId.toString(),
            '","description":"', w.content,
            '","author":"', Strings.toHexString(uint256(uint160(w.author)), 20),
            '","timestamp":', w.timestamp.toString(),
            ',"likes":', w.likes.toString(),
            ',"image":"https://via.placeholder.com/600x400/1a1a2e/ffffff?text=WordWall"}'
        ));
    }

    /// @notice Withdraw funds
    function withdraw() external onlyOwner {
        (bool success, ) = payable(owner()).call{value: address(this).balance}("");
        require(success, "Withdraw failed");
    }

    // Internal check if token exists
    function _exists(uint256 tokenId) internal view returns (bool) {
        return _ownerOf(tokenId) != address(0);
    }
}
