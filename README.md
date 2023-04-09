## 环境准备

- Rust
- Anchor（通过 AVM 安装）
- Solana-CLI

## 开始

### 初始化项目

使用 Anchor 初始化项目：

```bash
anchor init solana-hello-world
```

Anchor 最新版是 `0.27.0` 但 build 时会报错 rustc 版本过低，所以将 `programs/solana-hello-world/Cargo.toml` 中的 anchor 版本降为 `0.25.0`

```toml
[dependencies]
anchor-lang = "0.25.0"
```

### 编写合约(program)代码

将 `programs/solana-hello-world/src/lib.rs` 中内容替换为：

```rust
use anchor_lang::prelude::*;

declare_id!("HVLU57RJJ9hHXpnko3LDcEhfRUDuDUxx3JiHvibm59Dw");

#[program]
pub mod solana_hello_world {
    use super::*;

    pub fn create_message(ctx: Context<CreateMessage>, content: String) -> Result<()> {
        let message: &mut Account<Message> = &mut ctx.accounts.message;
        let author: &Signer = &ctx.accounts.author;
        let clock: Clock = Clock::get().unwrap();
        
        message.author = *author.key;
        message.timestamp = clock.unix_timestamp;
        message.content = content;
        
        Ok(())
    }

    pub fn update_message(ctx: Context<UpdateMessage>, content: String) -> Result<()> {
        let message: &mut Account<Message> = &mut ctx.accounts.message;
        let author: &Signer = &ctx.accounts.author;
        let clock: Clock = Clock::get().unwrap();
        
        message.author = *author.key;
        message.timestamp = clock.unix_timestamp;
        message.content = content;
        
        Ok(())
    }
}

#[derive(Accounts)]
pub struct CreateMessage<'info> {
        #[account(init, payer = author, space = 1000)]
    pub message: Account<'info, Message>,
        #[account(mut)]
    pub author: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct UpdateMessage<'info> {
        #[account(mut)]
    pub message: Account<'info, Message>,
        #[account(mut)]
    pub author: Signer<'info>,
}

#[account]
pub struct Message {
    pub author: Pubkey,
    pub timestamp: i64,
    pub content: String,
}
```

保存后开始编译：

```shell
anchor build
```

编译成功后会出现 `target/deploy/solana_hello_world.so` 文件，它是用于部署的 bpf 字节码

> 教程说编译后就会出现 program id 但我尝试发现是在部署后。。。

### 部署

新建一个独立 terminal 运行 `solana-test-validator` ，这会启动一个本地节点以供后面部署（这个终端结束前不要关闭）

将合约部署上链：

```shell
anchor deploy
```

部署成功后会提示：`Program ID: HVLU57RJJ9hHXpnko3LDcEhfRUDuDUxx3JiHvibm59Dw` 

将提示的 Program ID 复制下来粘贴到 `lib.rs` 和 `Anchor.toml` 对应的字段

```rust
declare_id!("HVLU57RJJ9hHXpnko3LDcEhfRUDuDUxx3JiHvibm59Dw");
```

```toml
[programs.localnet]
solana_hello_world = "HVLU57RJJ9hHXpnko3LDcEhfRUDuDUxx3JiHvibm59Dw"
```

保存后重新编译：

```shell
anchor build
```

编译成功说明没问题

### 测试

修改 `test/solana-hello-world.ts` 的内容为：

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { SolanaHelloWorld } from "../target/types/solana_hello_world";
import * as assert from "assert";

describe("solana-hello-world", () => {
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);

  const program = anchor.workspace
    .SolanaHelloWorld as Program<SolanaHelloWorld>;

  it("Can create a message", async () => {
    const message = anchor.web3.Keypair.generate();
    const messageContent = "Hello World!";
    await program.rpc.createMessage(messageContent, {
      accounts: {
        message: message.publicKey,
        author: provider.wallet.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      },
      signers: [message],
    });

    const messageAccount = await program.account.message.fetch(
      message.publicKey
    );

    assert.equal(
      messageAccount.author.toBase58(),
      provider.wallet.publicKey.toBase58()
    );
    assert.equal(messageAccount.content, messageContent);
    assert.ok(messageAccount.timestamp);
  });

  it("Can create and then update a message", async () => {
    const message = anchor.web3.Keypair.generate();
    const messageContent = "Hello World!";
    await program.rpc.createMessage(messageContent, {
      accounts: {
        message: message.publicKey,
        author: provider.wallet.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      },
      signers: [message],
    });

    const updatedMessageContent = "Solana is cool!";
    await program.rpc.updateMessage(updatedMessageContent, {
      accounts: {
        message: message.publicKey,
        author: provider.wallet.publicKey,
      },
    });

    const messageAccount = await program.account.message.fetch(
      message.publicKey
    );

    assert.equal(
      messageAccount.author.toBase58(),
      provider.wallet.publicKey.toBase58()
    );
    assert.notEqual(messageAccount.content, messageContent);
    assert.equal(messageAccount.content, updatedMessageContent);
    assert.ok(messageAccount.timestamp);
  });
});
```

保存后直接运行 `anchor test` 会报错 `Error: Your configured rpc port: 8899 is already in use`

不太清楚为什么报错，但可以通过指定 `provider.cluster` 来解决

```shell
anchor test --provider.cluster http://127.0.0.1:8899
```

IP和端口要和 `solana-test-validator` 的对齐

测试通过就结束了

## 小结

目前还不太看得懂 Solana on-chain Program，也因此不太会测试，希望学完 Rust 后能进步些

很多 bug 都是由于版本过新导致，要善于降级
