# Hướng dẫn viết Test với Jest (NestJS)

## 1. Jest là gì?

Jest là **test runner** — công cụ tự động: tìm file test → chạy code → so sánh kết quả thực tế với kết quả mong đợi → in báo cáo pass/fail.

**Luồng hoạt động khi chạy `npm run test`:**

```
1. Jest quét project, tìm mọi file kết thúc bằng .spec.ts hoặc .test.ts
2. Với mỗi file, Jest chạy trong 1 môi trường riêng biệt (sandbox)
3. Trong file đó, Jest thực thi các khối describe/it theo thứ tự viết
4. Mỗi assertion (expect...) được kiểm tra — đúng thì pass, sai thì fail
5. Jest tổng hợp kết quả, in ra terminal (xanh = pass, đỏ = fail)
```

---

## 2. Cấu trúc `describe` / `it`

```ts
describe('UsersService', () => {        // nhóm test (test suite)
  describe('findOne', () => {           // có thể lồng nhóm con
    it('should return a user if found', async () => {  // 1 test case cụ thể
      // ...
    });
  });
});
```

- `describe()` — chỉ để **nhóm** test lại cho dễ đọc report, không ảnh hưởng logic chạy
- `it()` (hoặc `test()`, tương đương nhau) — 1 test case thực sự chạy và được đánh giá pass/fail riêng biệt

Report hiển thị dạng cây:
```
UsersService
  ✓ should be defined
  findOne
    ✓ should return a user if found
    ✓ should throw NotFoundException if user not found
```

---

## 3. `expect()` và Matchers

```ts
expect(result).toEqual(mockUser);
```

- `expect(giá_trị_thực_tế)` → trả về object có nhiều "matcher"
- Matcher → kiểm tra 1 kiểu so sánh cụ thể

| Matcher | Ý nghĩa |
|---|---|
| `toBe(x)` | So sánh y hệt (`===`), dùng cho primitive (số, string) |
| `toEqual(x)` | So sánh giá trị sâu (deep equal), dùng cho object/array |
| `toThrow(Error)` | Kiểm tra hàm có throw đúng loại lỗi không |
| `toHaveBeenCalledWith(...)` | Kiểm tra mock function có được gọi với đúng tham số không |
| `toHaveBeenCalled()` | Kiểm tra mock function có được gọi hay không (không quan tâm tham số) |
| `toBeDefined()` | Kiểm tra giá trị khác `undefined` |
| `toBeNull()` | Kiểm tra giá trị là `null` |
| `toBeGreaterThan(x)` | So sánh số lớn hơn |

Nếu matcher fail, Jest in ra rõ **giá trị mong đợi vs giá trị thực tế** để debug.

---

## 4. Mocking

```ts
const mockUserModel = {
  findById: jest.fn(),
};
```

`jest.fn()` tạo ra **hàm giả** (mock function):
- Không chạy logic thật
- Ghi nhớ: đã được gọi bao nhiêu lần, với tham số gì
- Cho phép định nghĩa trước kết quả trả về:

```ts
mockUserModel.findById.mockResolvedValue(mockUser);
// mỗi khi gọi findById(...), tự động trả về Promise.resolve(mockUser)
```

**Vì sao cần mock?**
Service phụ thuộc vào Model (Mongoose) để gọi DB thật. Trong unit test, không muốn gọi DB thật (chậm, không ổn định, cần MongoDB đang chạy) → "đánh lừa" Service bằng object giả có cùng hình dạng nhưng hành vi do mình tự kiểm soát.

```ts
const module: TestingModule = await Test.createTestingModule({
  providers: [
    UsersService,
    { provide: getModelToken(User.name), useValue: mockUserModel }, // thay Model thật bằng mock
  ],
}).compile();
```

→ NestJS Dependency Injection giúp "tiêm" mock vào thay cho dependency thật.

---

## 5. Lifecycle Hooks

```ts
beforeEach(async () => {
  // chạy TRƯỚC mỗi it() trong describe này
});

afterEach(() => {
  // chạy SAU mỗi it()
});

beforeAll(() => {
  // chạy 1 LẦN DUY NHẤT trước tất cả it() trong describe này
});

afterAll(() => {
  // chạy 1 LẦN DUY NHẤT sau tất cả it()
});
```

Ví dụ thực tế:
```ts
beforeEach(async () => {
  const module = await Test.createTestingModule({...}).compile();
  service = module.get<UsersService>(UsersService);
  jest.clearAllMocks();  // reset "bộ nhớ" của mock trước mỗi test
});
```

→ Đảm bảo mỗi test case chạy **độc lập**, không bị ảnh hưởng bởi test trước đó.

---

## 6. Xử lý bất đồng bộ (async)

```ts
it('should return a user if found', async () => {
  mockUserModel.findById.mockResolvedValue(mockUser);
  const result = await service.findOne(mockUser._id);  // phải await
  expect(result).toEqual(mockUser);
});
```

Jest **chờ** Promise resolve xong mới đánh giá kết quả. Nếu quên `await`, test có thể pass giả (false positive).

**Test lỗi bất đồng bộ (Promise bị reject):**
```ts
await expect(service.findOne('not-exist-id'))
  .rejects.toThrow(NotFoundException);
```
`rejects` báo Jest: đang test 1 Promise sẽ bị reject (throw lỗi), hãy chờ và kiểm tra lỗi đó.

---

## 7. Ví dụ đầy đủ — phân tích từng bước

```ts
it('should throw NotFoundException if user not found', async () => {
  mockUserModel.findById.mockResolvedValue(null);           // B1: setup mock trả về null
  await expect(service.findOne('not-exist-id'))              // B2: gọi hàm thật cần test
    .rejects.toThrow(NotFoundException);                     // B3: kiểm tra kết quả đúng mong đợi
});
```

1. `beforeEach` chạy trước → tạo lại `service` sạch, reset mock
2. Setup mock trả về `null` (giả lập "không tìm thấy user trong DB")
3. Gọi `service.findOne(...)` — code thật chạy, vì `findById()` trả `null` nên `if (!user) throw new NotFoundException()` được kích hoạt
4. Jest kiểm tra: đúng có exception `NotFoundException` được throw không → pass/fail

---

## 8. Các loại test trong NestJS

| Loại test | Test cái gì | Có gọi DB/HTTP thật không |
|---|---|---|
| **Unit test (Service)** | Logic nghiệp vụ trong Service, mock Model | Không |
| **Unit test (Controller)** | Controller có gọi đúng method Service không | Không |
| **DTO validation test** | Rule validate của DTO (class-validator) | Không |
| **E2E test** | Toàn bộ luồng HTTP thật → DB thật | Có (cần MongoDB đang chạy) |

**Ưu tiên viết test theo mô hình Testing Pyramid:**
- Service test → nhiều nhất (logic phức tạp, dễ sai)
- Controller test → vừa phải (thường chỉ gọi service)
- E2E test → ít, chỉ test luồng quan trọng nhất

---

## 9. Các lệnh chạy Jest thường dùng

```bash
npm run test              # chạy tất cả unit test 1 lần
npm run test:watch        # watch mode — tự chạy lại khi sửa code
npm run test:cov          # xuất báo cáo coverage (% code được test)
npm run test:e2e          # chạy e2e test (thư mục test/)
npm run test -- users     # chỉ chạy file test có tên chứa "users"
```

**Phím tắt trong watch mode:**
- `a` → chạy lại toàn bộ test
- `f` → chỉ chạy lại test đang fail
- `p` → lọc theo tên file
- `t` → lọc theo tên test case
- `q` → thoát

---

## 10. Ví dụ file test hoàn chỉnh (Service)

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { getModelToken } from '@nestjs/mongoose';
import { NotFoundException } from '@nestjs/common';
import { UsersService } from './users.service';
import { User } from './schemas/user.schema';

const mockUser = {
  _id: '64b7f9f9f9f9f9f9f9f9f9f9',
  name: 'Nam',
  email: 'nam@test.com',
  age: 20,
};

const mockUserModel = {
  create: jest.fn(),
  find: jest.fn(),
  findById: jest.fn(),
  findByIdAndUpdate: jest.fn(),
  findByIdAndDelete: jest.fn(),
};

describe('UsersService', () => {
  let service: UsersService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        { provide: getModelToken(User.name), useValue: mockUserModel },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
    jest.clearAllMocks();
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  describe('findOne', () => {
    it('should return a user if found', async () => {
      mockUserModel.findById.mockResolvedValue(mockUser);
      const result = await service.findOne(mockUser._id);
      expect(result).toEqual(mockUser);
    });

    it('should throw NotFoundException if user not found', async () => {
      mockUserModel.findById.mockResolvedValue(null);
      await expect(service.findOne('not-exist-id'))
        .rejects.toThrow(NotFoundException);
    });
  });
});
```

---

## 11. Ghi nhớ nhanh (cheat sheet)

| Muốn làm gì | Viết gì |
|---|---|
| Nhóm test lại | `describe('...', () => {})` |
| 1 test case | `it('...', () => {})` hoặc `test('...', () => {})` |
| Kiểm tra giá trị | `expect(x).toBe(y)` / `toEqual(y)` |
| Tạo hàm giả | `jest.fn()` |
| Định nghĩa mock trả về gì | `.mockResolvedValue(x)` (async) / `.mockReturnValue(x)` (sync) |
| Reset mock | `jest.clearAllMocks()` |
| Setup trước mỗi test | `beforeEach(() => {})` |
| Test hàm async | `await expect(promise).resolves.toBe(x)` |
| Test lỗi async | `await expect(promise).rejects.toThrow(Error)` |