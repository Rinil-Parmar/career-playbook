# Lesson 1: Class, Object, Encapsulation

## What to Understand

- **Class**: blueprint for behavior and data
- **Object**: runtime instance of a class
- **Encapsulation**: hide internals and expose safe operations

In industry code, encapsulation protects invariants and avoids invalid state changes.

## Example: `BankAccount`

```java
class BankAccount {
    private String accountHolder;
    private double balance;

    public BankAccount(String accountHolder, double initialBalance) {
        this.accountHolder = accountHolder;
        this.balance = initialBalance;
    }

    public void deposit(double amount) {
        if (amount > 0) {
            balance += amount;
        }
    }

    public void withdraw(double amount) {
        if (amount > 0 && amount <= balance) {
            balance -= amount;
        }
    }

    public String getAccountHolder() {
        return accountHolder;
    }

    public double getBalance() {
        return balance;
    }
}

public class Main {
    public static void main(String[] args) {
        BankAccount acc = new BankAccount("Parmar", 1000);
        acc.deposit(500);
        acc.withdraw(300);
        System.out.println(acc.getAccountHolder() + " balance: " + acc.getBalance());
    }
}
```

## Why This Is Industry Standard

1. `balance` is `private` so it cannot be changed directly from outside.
2. Business rules live in methods (`deposit`, `withdraw`) where input is controlled.
3. The class owns its state and enforces valid transitions.

## Interview Angle

**Common question:** Why not make all fields public for easier coding?

**Strong answer:** Public fields break encapsulation and allow invalid state changes. With private fields and controlled methods, you keep data valid, reduce bugs, and preserve backward compatibility.

## Coding Prompts

1. Add a transfer method: `transferTo(BankAccount target, double amount)`.
2. Prevent creating an account with negative initial balance.
3. Add a `toString()` method for logging-friendly output.
