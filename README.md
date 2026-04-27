// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Election {
    struct Candidate {
        uint id;
        string name;
        uint voteCount;
    }

    mapping(address => bool) public voters;
    mapping(uint => Candidate) public candidates;
    uint public candidatesCount;

    event votedEvent(uint indexed _candidateId);

    constructor() {
        addCandidate("Alice - Đội A");
        addCandidate("Bob - Đội B");
    }

    function addCandidate(string memory _name) private {
        candidatesCount++;
        candidates[candidatesCount] = Candidate(candidatesCount, _name, 0);
    }

    function vote(uint _candidateId) public {
        require(!voters[msg.sender], "Loi: Tai khoan nay da bo phieu.");
        require(_candidateId > 0 && _candidateId <= candidatesCount, "Loi: Ung cu vien khong hop le.");

        voters[msg.sender] = true;
        candidates[_candidateId].voteCount++;

        emit votedEvent(_candidateId);
    }
}




const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("Hệ thống Bầu cử (Election Contract)", function () {
    let election;
    let owner, addr1, addr2;

    beforeEach(async function () {
        [owner, addr1, addr2] = await ethers.getSigners();
        const Election = await ethers.getContractFactory("Election");
        election = await Election.deploy();
    });

    it("1. Khởi tạo với đúng 2 ứng cử viên", async function () {
        expect(await election.candidatesCount()).to.equal(2);
    });

    it("2. Dữ liệu của ứng cử viên ban đầu phải chính xác", async function () {
        const candidate1 = await election.candidates(1);
        expect(candidate1.id).to.equal(1);
        expect(candidate1.name).to.equal("Alice - Đội A");
        expect(candidate1.voteCount).to.equal(0);
    });

    it("3. Cho phép tài khoản hợp lệ bỏ phiếu và emit sự kiện", async function () {
        await expect(election.connect(addr1).vote(1))
            .to.emit(election, "votedEvent")
            .withArgs(1);

        expect(await election.voters(addr1.address)).to.be.true;

        const candidate1 = await election.candidates(1);
        expect(candidate1.voteCount).to.equal(1);
    });

    it("4. Báo lỗi khi bỏ phiếu cho ứng cử viên không hợp lệ", async function () {
        await expect(election.connect(addr1).vote(99)).to.be.revertedWith("Loi: Ung cu vien khong hop le.");
    });

    it("5. Báo lỗi khi một tài khoản cố tình bầu 2 lần (Double-voting)", async function () {
        await election.connect(addr2).vote(2);
        await expect(election.connect(addr2).vote(2)).to.be.revertedWith("Loi: Tai khoan nay da bo phieu.");
    });
});






const hre = require("hardhat");

async function main() {
  // Lấy bản thiết kế của Contract
  const Election = await hre.ethers.getContractFactory("Election");
  
  // Tiến hành deploy
  const election = await Election.deploy();

  // Đợi giao dịch deploy được xác nhận (Ethers v6)
  await election.waitForDeployment();

  console.log(`Hợp đồng Election đã được deploy thành công tại địa chỉ: ${await election.getAddress()}`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
