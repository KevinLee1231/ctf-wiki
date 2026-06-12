# RePairing

## 题目简述

加密结构是 pairing/ElGamal 风格的可重随机化密文：`C1 = shared_key * pk^t`、`C2 = G^t`、`C3 = h1(id)^t`。服务端没有检测等价密文，导致可提交重随机化后的密文让服务端返回同一 shared key。

## 解题过程

### 关键观察

加密结构是 pairing/ElGamal 风格的可重随机化密文：`C1 = shared_key * pk^t`、`C2 = G^t`、`C3 = h1(id)^t`。

### 求解步骤

enc is ElGamal-style: C1 = shared_key · pk^t, C2 = G^t, C3 = h1(id)^t. Because the scheme is
multiplicatively homomorphic, adding a random scalar r to t via (C1·pk^r, C2+G·r, C3+h1(id)·r)
produces a fresh-looking ciphertext that decrypts to the same shared secret. The server fails to
detect this re-randomization, so it happily decrypts our forged ciphertext and returns the very
key it used for the original flag encryption.

use std::env;
use std::io::{BufRead, BufReader, Write};
use std::net::TcpStream;

use anyhow::{Context, Result, bail};
use ark_bls12_381::{Fr, G1Projective, G2Projective};
use ark_ec::PrimeGroup;
use ark_ff::{Field, PrimeField, Zero};
use ark_std::UniformRand;
use common::{hex_g1, hex_g2, hex_gt, parse_g1, parse_g2, parse_gt};
use rand::rngs::OsRng;

fn random_non_zero_scalar() -> Fr {
    let mut rng = OsRng;
    loop {
        let candidate = Fr::rand(&mut rng);
        if !candidate.is_zero() {
            return candidate;
        }
    }
}

fn rerandomize(
    c1: &common::GT,
    c2: &G1Projective,
    c3: &G2Projective,
    pk: &common::GT,
    q: &G2Projective,
    r: Fr,
) -> (common::GT, G1Projective, G2Projective) {
    let pk_pow = pk.pow(r.into_bigint());
    let new_c1 = c1.clone() * pk_pow;
    let new_c2 = c2.clone() + G1Projective::generator() * r;
    let new_c3 = c3.clone() + q.clone() * r;
    (new_c1, new_c2, new_c3)
}

fn main() -> Result<()> {
    let addr = env::args()
        .nth(1)
        .unwrap_or_else(|| "<challenge-host>:42601".to_string());

    let mut stream = TcpStream::connect(&addr).with_context(|| format!
("connect to {addr}"))?;
    stream.set_nodelay(true).context("enable TCP_NODELAY")?;
    let mut reader = BufReader::new(stream.try_clone().context("clone
stream")?);

    let mut banner = String::new();
    reader.read_line(&mut banner).context("read banner")?;
    let banner = banner.trim();
    if banner.is_empty() {
        bail!("empty banner");
    }

    let mut parts = banner.split('|');
    let id_hex = parts.next().context("missing id")?;
    let _dst_hex = parts.next().context("missing DST")?;
    let pk_hex = parts.next().context("missing pk")?;
    let q_hex = parts.next().context("missing h1(id)")?;
    let c1_hex = parts.next().context("missing c1")?;
    let c2_hex = parts.next().context("missing c2")?;
    let c3_hex = parts.next().context("missing c3")?;
    let enc_hex = parts.next().context("missing ciphertext")?;
    if parts.next().is_some() {
        bail!("unexpected extra data in banner");
    }

    let id_bytes = hex::decode(id_hex).context("decode id")?;
    let id = String::from_utf8(id_bytes).context("id utf8")?;

    let pk = parse_gt(pk_hex).context("parse pk")?;
    let q = parse_g2(q_hex).context("parse q")?;
    let c1 = parse_gt(c1_hex).context("parse c1")?;
    let c2 = parse_g1(c2_hex).context("parse c2")?;
    let c3 = parse_g2(c3_hex).context("parse c3")?;
    let encrypted_flag = hex::decode(enc_hex).context("decode encrypted
flag")?;

    let r = random_non_zero_scalar();
    let (new_c1, new_c2, new_c3) = rerandomize(&c1, &c2, &c3, &pk, &q, r);

    let payload = format!(
        "{}|{}|{}\\n",
        hex_gt(&new_c1),
        hex_g1(&new_c2),
        hex_g2(&new_c3)
    );

    stream
        .write_all(payload.as_bytes())
        .context("send forged ciphertext")?;
    stream.flush().context("flush")?;

### 跨页补回：重随机化脚本收尾

let mut response = String::new();
    reader.read_line(&mut response).context("read key")?;
    let key_bytes = hex::decode(response.trim()).context("decode key")?;

    if key_bytes.len() != encrypted_flag.len() {
        bail!("key length mismatch");
    }

    let flag: Vec<u8> = encrypted_flag
        .iter()
        .zip(key_bytes.iter())
        .map(|(a, b)| a ^ b)
        .collect();

    println!("{id}: {}", String::from_utf8_lossy(&flag));

    Ok(())
}

## 方法总结

- 核心技巧：ElGamal 类密文重随机化绕过去重。
- 识别信号：密文分量可同时乘/加同一随机量而保持明文 shared key 不变。
- 复用要点：如果服务端给解密 oracle，先检查密文是否绑定原始随机数或唯一性。
