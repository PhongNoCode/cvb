Tuyệt vời, chúng ta sẽ bắt tay vào làm thực tế. Hãy mở Terminal (WSL2, macOS Terminal hoặc Git Bash trên Windows) và thực hiện tuần tự theo từng bước dưới đây.

Bước 1: Khởi tạo thư mục và dự án
Đầu tiên, tạo thư mục cho dự án, khởi tạo Git và Node.js. Chạy từng dòng lệnh sau:

Bash
mkdir voting-dapp-backend
cd voting-dapp-backend
git init
npm init -y
Bước 2: Cài đặt Hardhat và các công cụ
Cài đặt Hardhat và gói toolbox chứa đầy đủ các công cụ như Ethers.js v6, Mocha, Chai.

Bash
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox
Tiếp theo, khởi tạo cấu trúc dự án Hardhat:

Bash
npx hardhat init
Lưu ý khi menu hiện ra:

Chọn "Create a JavaScript project" (nhấn Enter).

Nhấn Enter liên tục 3 lần nữa để chấp nhận các thư mục mặc định (Hardhat project root, thêm .gitignore, và cài đặt dependencies).

Bước 3: Cài đặt Smart Contract
Hardhat đã tạo sẵn một file mẫu là contracts/Lock.sol. Bạn hãy xóa file đó đi và tạo một file mới tên là contracts/Election.sol, sau đó dán đoạn code sau vào:

Solidity
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
Bước 4: Viết kịch bản kiểm thử (Test Cases)
Tương tự, xóa file test mẫu test/Lock.js và tạo file mới test/Election.js. Dán đoạn mã dùng Mocha + Chai này vào:

JavaScript
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
Chạy lệnh sau để biên dịch và kiểm tra:

Bash
npx hardhat test
Nếu thấy hiện ra 5 dòng tích xanh (passing), chúc mừng bạn, Contract đã hoạt động hoàn hảo!

Bước 5: Viết Script Deploy (Triển khai hợp đồng)
Tạo một thư mục mới tên là scripts ở thư mục gốc của dự án, và tạo file scripts/deploy.js bên trong đó:

JavaScript
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
Bước 6: Chạy Blockchain Local và Deploy
Hardhat có tích hợp sẵn một mạng lưới tương đương với Ganache.

Mở một cửa sổ Terminal mới (giữ nguyên cửa sổ cũ) và chạy lệnh này để bật blockchain local:

Bash
npx hardhat node
(Nó sẽ in ra danh sách 20 tài khoản và Private Key. Cứ treo cửa sổ này ở đó).

Quay lại cửa sổ Terminal cũ, chạy lệnh deploy mạng lưới local vừa tạo:

Bash
npx hardhat run scripts/deploy.js --network localhost
(Bạn sẽ nhận được địa chỉ của Contract, ví dụ: 0x5FbDB2315678afecb367f032d93F642f64180aa3. Hãy lưu địa chỉ này lại nếu sau này làm tiếp phần Frontend).

Bước 7: Đẩy mã nguồn lên GitHub (Bắt buộc theo yêu cầu của bạn)
Cuối cùng, lưu trữ thành quả của bạn lên GitHub.

Đăng nhập vào GitHub và tạo một Repository mới có tên voting-dapp.

Chạy các lệnh sau trong Terminal (thay link repo của bạn vào dòng thứ 4):

Bash
git add .
git commit -m "Hoan thien Smart Contract, Test va Deploy Script"
git branch -M main
git remote add origin https://github.com/TÊN_USER_CỦA_BẠN/voting-dapp.git
git push -u origin main
