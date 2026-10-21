import java.util.Scanner;
import java.util.regex.Pattern;
import java.util.Objects;
import java.lang.annotation.*;
import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.List;
import java.util.HashMap;
import java.util.Comparator;
import java.util.Map;
import java.util.Collections;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.io.*;
import java.nio.file.*;
import java.nio.file.attribute.BasicFileAttributes;

class BankException extends Exception {
    public BankException(String message) {
        super(message);
    }
}

class InsufficientFundsException extends BankException {
    private long shortfall;

    public InsufficientFundsException(String message, long shortfall) {
        super(message);
        this.shortfall = shortfall;
    }

    public long getShortfall() {
        return shortfall;
    }
}

class AccountNotFoundException extends BankException {
    public AccountNotFoundException(String message) {
        super(message);
    }
}

class InvalidAmountException extends BankException {
    public InvalidAmountException(String message) {
        super(message);
    }
}

record BankInfo(String name, String branch) {}

enum MenuOption {
    OPEN_ACCOUNT, DEPOSIT, WITHDRAW, TRANSFER, STATEMENT, LIST_ACCOUNTS,
    BATCH_PROCESSING, REPORT, EXIT
}

enum TransactionType {
    DEPOSIT, WITHDRAW, TRANSFER
}

record Command(TransactionType type, String accountNumber, long amount) {}

interface Transactable {
    void deposit(long amount) throws InvalidAmountException;

    boolean withdraw(long amount)
            throws InsufficientFundsException, InvalidAmountException;
}

interface InterestBearing {
    default double yearlyInterest() {
        Account account = (Account) this;
        return account.getBalance() * account.interestRate() / 100;
    }
}

@FunctionalInterface
interface WithdrawRule {
    boolean allow(Account account, long amount);
}

interface Premium {}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Id {}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Positive {
    String message() default "must be > 0";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface MaxLength {
    int value();
}

class Customer implements Cloneable, Serializable {
    private static final long serialVersionUID = 1L;

    private String name;
    private String email;
    private String mobile;
    private final String customerId;
    private static long customerCounter = 101;
    private Address address;

    private static String generateCustomerId() {
        String id = "CUST" + customerCounter;
        customerCounter++;
        return id;
    }

    Customer(String name, String email, String mobile) {
        this.name = name;
        this.email = email;
        this.mobile = mobile;
        customerId = generateCustomerId();
    }

    public String getName() {
        return name;
    }

    public String getEmail() {
        return email;
    }

    public String getMobile() {
        return mobile;
    }

    public String getCustomerId() {
        return customerId;
    }

    public static class Address implements Serializable {
        private static final long serialVersionUID = 1L;

        private String line;
        private String city;
        private String pincode;

        Address(String line, String city, String pincode) {
            this.line = line;
            this.city = city;
            this.pincode = pincode;
        }

        public String getLine() {
            return line;
        }

        public String getCity() {
            return city;
        }

        public String getPincode() {
            return pincode;
        }
    }

    public void setAddress(Address address) {
        this.address = address;
    }

    public Address getAddress() {
        return address;
    }

    @Override
    public Customer clone() {
        try {
            return (Customer) super.clone();
        } catch (CloneNotSupportedException e) {
            return null;
        }
    }
}

abstract class Account
        implements Transactable, InterestBearing, Serializable, Comparable<Account> {
    private static final long serialVersionUID = 1L;

    @Id
    private final String accountNumber;

    @MaxLength(30)
    private String ownerName;

    @Positive
    private long balance;

    private boolean active;

    private static int accountCounter = 1;

    private static synchronized String generateAccountNumber() {
        String number = String.format("AC%04d", accountCounter);
        accountCounter++;
        return number;
    }

    static synchronized void ensureCounterAbove(String accountNumber) {
        if (accountNumber != null && accountNumber.matches("AC[0-9]{4,}")) {
            try {
                int existing = Integer.parseInt(accountNumber.substring(2));
                if (existing >= accountCounter) accountCounter = existing + 1;
            } catch (NumberFormatException ignored) {
                // Keep the current counter if the identifier is outside integer range.
            }
        }
    }

    Account(String ownerName, long openingBalance) {
        accountNumber = generateAccountNumber();
        this.ownerName = ownerName;
        balance = openingBalance;
        active = true;
    }

    Account(String ownerName) {
        this(ownerName, 0);
    }

    @Override
    public synchronized void deposit(long amount) throws InvalidAmountException {
        if (amount <= 0) {
            throw new InvalidAmountException("Deposit amount must be greater than zero.");
        }
        balance = balance + amount;
    }

    public void unsafeDeposit(long amount) throws InvalidAmountException {
        if (amount <= 0) {
            throw new InvalidAmountException("Deposit amount must be greater than zero.");
        }
        long temp = balance;
        Thread.yield();
        balance = temp + amount;
    }

    @Override
    public synchronized boolean withdraw(long amount)
            throws InsufficientFundsException, InvalidAmountException {
        if (amount <= 0) {
            throw new InvalidAmountException("Withdrawal amount must be greater than zero.");
        }
        if (!canWithdraw(amount)) {
            long shortfall = amount - getBalance();
            if (shortfall < 0) {
                shortfall = 0;
            }
            throw new InsufficientFundsException(
                    "Insufficient funds. Shortfall: " + shortfall, shortfall);
        }
        balance = balance - amount;
        return true;
    }

    public void transfer(Account to, long amount) throws BankException {
        try {
            if (to == null) {
                throw new AccountNotFoundException("Destination account not found.");
            }
            withdraw(amount);
            to.deposit(amount);
        } catch (BankException e) {
            throw e;
        } finally {
            System.out.println("Transfer attempt completed.");
        }
    }

    public abstract double interestRate();

    public abstract boolean canWithdraw(long amount);

    public String getAccountNumber() {
        return accountNumber;
    }

    public String getOwnerName() {
        return ownerName;
    }

    public long getBalance() {
        return balance;
    }

    public boolean isActive() {
        return active;
    }

    @Override
    public String toString() {
        return "Account Number: " + accountNumber
                + ", Owner: " + ownerName
                + ", Balance: " + balance;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Account)) return false;
        Account a = (Account) o;
        return accountNumber.equals(a.accountNumber);
    }

    @Override
    public int hashCode() {
        return Objects.hash(accountNumber);
    }

    @Override
    public int compareTo(Account other) {
        return this.accountNumber.compareTo(other.accountNumber);
    }
}

class SavingsAccount extends Account implements Serializable {
    private static final long serialVersionUID = 1L;
    private long minBalance;

    SavingsAccount(String ownerName, long openingBalance, long minBalance) {
        super(ownerName, openingBalance);
        this.minBalance = minBalance;
    }

    @Override
    public double interestRate() {
        return 4.0;
    }

    @Override
    public boolean canWithdraw(long amount) {
        return getBalance() - amount >= minBalance;
    }

    public long getMinBalance() {
        return minBalance;
    }
}

class CurrentAccount extends Account implements Serializable {
    private static final long serialVersionUID = 1L;
    private long overdraftLimit;

    CurrentAccount(String ownerName, long openingBalance, long overdraftLimit) {
        super(ownerName, openingBalance);
        this.overdraftLimit = overdraftLimit;
    }

    @Override
    public double interestRate() {
        return 0;
    }

    @Override
    public boolean canWithdraw(long amount) {
        return getBalance() - amount >= -overdraftLimit;
    }

    public long getOverdraftLimit() {
        return overdraftLimit;
    }
}

class FixedDepositAccount extends Account implements Serializable {
    private static final long serialVersionUID = 1L;

    FixedDepositAccount(String ownerName, long openingBalance) {
        super(ownerName, openingBalance);
    }

    @Override
    public double interestRate() {
        return 7.0;
    }

    @Override
    public boolean canWithdraw(long amount) {
        return false;
    }
}

class Validator {
    public static boolean isValidName(String name) {
        return name != null && !name.trim().isEmpty() && name.trim().length() <= 30;
    }

    public static boolean isValidAmount(long amount) {
        return amount > 0;
    }

    private static final Pattern MOBILE = Pattern.compile("^[0-9]{10}$");
    private static final Pattern EMAIL =
            Pattern.compile("^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$");
    private static final Pattern PAN = Pattern.compile("^[A-Z]{5}[0-9]{4}[A-Z]$");
    private static final Pattern IFSC = Pattern.compile("^[A-Z]{4}0[A-Z0-9]{6}$");

    public static boolean isValidMobile(String mobile) {
        return MOBILE.matcher(mobile).matches();
    }

    public static boolean isValidEmail(String email) {
        return EMAIL.matcher(email).matches();
    }

    public static boolean isValidPan(String pan) {
        return PAN.matcher(pan).matches();
    }

    public static boolean isValidIfsc(String ifsc) {
        return IFSC.matcher(ifsc).matches();
    }
}

class CommandParser {
    public static Command parse(String line) {
        if (line == null || line.trim().isEmpty()) {
            throw new IllegalArgumentException("Command cannot be empty.");
        }
        String[] parts = line.trim().split("\\s+");
        if (parts.length != 3) {
            throw new IllegalArgumentException("Use: TYPE ACCOUNT_NUMBER AMOUNT");
        }
        TransactionType type = TransactionType.valueOf(parts[0].toUpperCase());
        String accountNumber = parts[1];
        long amount = Long.parseLong(parts[2]);
        return new Command(type, accountNumber, amount);
    }
}

class StatementFormatter {
    public static String buildStatement(Account account) {
        StringBuilder statement = new StringBuilder();
        statement.append("ACCOUNT STATEMENT\n");
        statement.append("------------------------------\n");
        statement.append("Account Number: ");
        statement.append(account.getAccountNumber());
        statement.append("\n");
        statement.append("Owner: ");
        statement.append(account.getOwnerName());
        statement.append("\n");
        statement.append("Balance: ");
        statement.append(account.getBalance());
        statement.append("\n");
        statement.append("Interest Rate: ");
        statement.append(account.interestRate());
        statement.append("%\n");
        statement.append("Yearly Interest: ");
        statement.append(account.yearlyInterest());
        statement.append("\n");
        statement.append("Active: ");
        statement.append(account.isActive());
        statement.append("\n");
        return statement.toString();
    }
}

class AnnotationValidator {
    public static String[] validate(Object obj) {
        List<String> errors = new ArrayList<>();
        Class<?> current = obj.getClass();
        while (current != null) {
            Field[] fields = current.getDeclaredFields();
            for (Field field : fields) {
                try {
                    field.setAccessible(true);
                    Object value = field.get(obj);
                    if (field.isAnnotationPresent(Positive.class)) {
                        Positive annotation = field.getAnnotation(Positive.class);
                        if (value instanceof Number) {
                            Number number = (Number) value;
                            if (number.doubleValue() <= 0) {
                                errors.add(field.getName() + " " + annotation.message());
                            }
                        }
                    }
                    if (field.isAnnotationPresent(MaxLength.class)) {
                        MaxLength annotation = field.getAnnotation(MaxLength.class);
                        if (value != null
                                && value.toString().length() > annotation.value()) {
                            errors.add(field.getName()
                                    + " must not exceed "
                                    + annotation.value()
                                    + " characters");
                        }
                    }
                } catch (Exception e) {
                    errors.add("Error checking " + field.getName());
                }
            }
            current = current.getSuperclass();
        }
        return errors.toArray(new String[0]);
    }
}

class BankLogger implements AutoCloseable {
    public BankLogger() {
        System.out.println("Bank logger opened.");
    }

    public void log(String message) {
        System.out.println("LOG: " + message);
    }

    @Override
    public void close() {
        System.out.println("Bank logger closed.");
    }
}

class AccountWorker implements Runnable {
    private Account account;
    private int times;
    private long amount;
    private boolean safe;
    private CountDownLatch latch;

    AccountWorker(Account account, int times, long amount, boolean safe, CountDownLatch latch) {
        this.account = account;
        this.times = times;
        this.amount = amount;
        this.safe = safe;
        this.latch = latch;
    }

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " started");
        try {
            latch.await();
            for (int i = 0; i < times; i++) {
                if (safe) {
                    account.deposit(amount);
                } else {
                    account.unsafeDeposit(amount);
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.out.println(Thread.currentThread().getName() + " interrupted");
        } catch (InvalidAmountException e) {
            System.out.println(e.getMessage());
        }
        System.out.println(Thread.currentThread().getName() + " finished");
    }
}

// Executes transaction tasks using a managed pool of four worker threads.
class TransactionProcessor {
    private final ExecutorService executor = Executors.newFixedThreadPool(4);
    private final ConcurrentHashMap<String, Account> sharedAccounts =
            new ConcurrentHashMap<>();

    public void addAccount(Account account) {
        sharedAccounts.put(account.getAccountNumber(), account);
    }

    public Account getAccount(String accountNumber) {
        return sharedAccounts.get(accountNumber);
    }

    public ConcurrentHashMap<String, Account> getSharedAccounts() {
        return sharedAccounts;
    }

    public void submit(Runnable task) {
        executor.execute(task);
    }

    public void stop() throws InterruptedException {
        executor.shutdown();
        if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
            executor.shutdownNow();
            executor.awaitTermination(30, TimeUnit.SECONDS);
        }
    }
}

// A Runnable transaction that performs a deposit or withdrawal.
class BankTransactionTask implements Runnable {
    private final Account account;
    private final TransactionType type;
    private final long amount;

    BankTransactionTask(Account account, TransactionType type, long amount) {
        this.account = account;
        this.type = type;
        this.amount = amount;
    }

    @Override
    public void run() {
        try {
            if (type == TransactionType.DEPOSIT) {
                account.deposit(amount);
                System.out.println(Thread.currentThread().getName()
                        + " deposited " + amount + " into "
                        + account.getAccountNumber()
                        + "; balance = " + account.getBalance());
            } else if (type == TransactionType.WITHDRAW) {
                account.withdraw(amount);
                System.out.println(Thread.currentThread().getName()
                        + " withdrew " + amount + " from "
                        + account.getAccountNumber()
                        + "; balance = " + account.getBalance());
            }
        } catch (BankException e) {
            System.out.println("Transaction failed: " + e.getMessage());
        }
    }
}

// Fixed-size array buffer for producer-consumer transaction tasks.
class TransactionBuffer {
    private final Runnable[] buffer;
    private int head = 0;
    private int tail = 0;
    private int count = 0;

    TransactionBuffer(int capacity) {
        buffer = new Runnable[capacity];
    }

    public synchronized void put(Runnable task) throws InterruptedException {
        while (count == buffer.length) {
            wait();
        }
        buffer[tail] = task;
        tail = (tail + 1) % buffer.length;
        count++;
        notifyAll();
    }

    public synchronized Runnable take() throws InterruptedException {
        while (count == 0) {
            wait();
        }
        Runnable task = buffer[head];
        buffer[head] = null;
        head = (head + 1) % buffer.length;
        count--;
        notifyAll();
        return task;
    }
}

// Demonstrates the unsafe locking order and the corrected ordered transfer.
class TransferLockDemo {
    private static final Runnable STOP_TASK = () -> {};

    public static void demonstrateDeadlock() throws InterruptedException {
        System.out.println("\n--- Deadlock demonstration (unsafe lock order) ---");
        Account accountA = new SavingsAccount("Deadlock A", 1000, 0);
        Account accountB = new SavingsAccount("Deadlock B", 1000, 0);
        CountDownLatch bothHoldFirstLock = new CountDownLatch(2);

        Thread threadAB = new Thread(() -> {
            synchronized (accountA) {
                bothHoldFirstLock.countDown();
                try {
                    bothHoldFirstLock.await();
                    synchronized (accountB) {
                        performTransfer(accountA, accountB, 100);
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }, "Transfer-A-to-B");

        Thread threadBA = new Thread(() -> {
            synchronized (accountB) {
                bothHoldFirstLock.countDown();
                try {
                    bothHoldFirstLock.await();
                    synchronized (accountA) {
                        performTransfer(accountB, accountA, 100);
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }, "Transfer-B-to-A");

        // Daemon threads let the demo program exit even though these threads
        // intentionally remain blocked in the deadlock.
        threadAB.setDaemon(true);
        threadBA.setDaemon(true);
        threadAB.start();
        threadBA.start();
        threadAB.join(300);
        threadBA.join(300);

        if (threadAB.isAlive() && threadBA.isAlive()) {
            System.out.println("Deadlock reproduced: each thread holds one "
                    + "account lock and waits for the other.");
        } else {
            System.out.println("The deadlock was not reproduced on this run.");
        }

        System.out.println("\n--- Fixed transfer (consistent lock order) ---");
        Account safeA = new SavingsAccount("Safe A", 1000, 0);
        Account safeB = new SavingsAccount("Safe B", 1000, 0);

        Thread safeAB = new Thread(
                () -> orderedTransfer(safeA, safeB, 100), "Safe-A-to-B");
        Thread safeBA = new Thread(
                () -> orderedTransfer(safeB, safeA, 100), "Safe-B-to-A");

        safeAB.start();
        safeBA.start();
        safeAB.join();
        safeBA.join();

        System.out.println("Safe transfer threads completed.");
        System.out.println(safeA.getAccountNumber() + " balance: " + safeA.getBalance());
        System.out.println(safeB.getAccountNumber() + " balance: " + safeB.getBalance());
    }

    private static void orderedTransfer(Account from, Account to, long amount) {
        Account first;
        Account second;
        if (from.getAccountNumber().compareTo(to.getAccountNumber()) < 0) {
            first = from;
            second = to;
        } else {
            first = to;
            second = from;
        }
        synchronized (first) {
            synchronized (second) {
                performTransfer(from, to, amount);
            }
        }
    }

    private static void performTransfer(Account from, Account to, long amount) {
        try {
            from.withdraw(amount);
            to.deposit(amount);
            System.out.println(Thread.currentThread().getName()
                    + " transferred " + amount + " from "
                    + from.getAccountNumber() + " to "
                    + to.getAccountNumber());
        } catch (BankException e) {
            System.out.println("Transfer failed: " + e.getMessage());
        }
    }
}

class AccountStore {
    private final HashMap<String, Account> accounts = new HashMap<>();
    private final ArrayList<String> transactionHistory = new ArrayList<>();

    public void add(Account account) {
        if (account == null) {
            throw new IllegalArgumentException("Account cannot be null.");
        }
        accounts.put(account.getAccountNumber(), account);
    }

    public Account get(String accountNumber) {
        return accounts.get(accountNumber);
    }

    public ArrayList<Account> getAll() {
        return new ArrayList<>(accounts.values());
    }

    public void addTransaction(String transaction) {
        transactionHistory.add(transaction);
    }

    public ArrayList<String> getTransactionHistory() {
        return new ArrayList<>(transactionHistory);
    }

    public ArrayList<Account> sortedByBalance() {
        ArrayList<Account> sortedAccounts = getAll();
        sortedAccounts.sort(Comparator.comparingLong(Account::getBalance));
        return sortedAccounts;
    }
}

class Repository<T> {
    private final ConcurrentHashMap<String, T> data = new ConcurrentHashMap<>();

    public T save(T entity) {
        if (entity == null) {
            throw new IllegalArgumentException("Entity cannot be null.");
        }
        Class<?> current = entity.getClass();
        while (current != null) {
            for (Field field : current.getDeclaredFields()) {
                if (field.isAnnotationPresent(Id.class)) {
                    try {
                        field.setAccessible(true);
                        Object value = field.get(entity);
                        if (value == null) {
                            throw new IllegalArgumentException("The @Id field cannot be null.");
                        }
                        data.put(value.toString(), entity);
                        return entity;
                    } catch (IllegalAccessException e) {
                        throw new RuntimeException("Could not read the @Id field.", e);
                    }
                }
            }
            current = current.getSuperclass();
        }
        throw new IllegalArgumentException(
                "No field marked with @Id found in " + entity.getClass().getName());
    }

    public T findById(String id) {
        return data.get(id);
    }

    public ArrayList<T> findAll() {
        return new ArrayList<>(data.values());
    }

    public boolean delete(String id) {
        return data.remove(id) != null;
    }

    @SafeVarargs
    public final void saveAll(T... items) {
        for (T item : items) {
            save(item);
        }
    }

    public static <N extends Number> double average(N[] values) {
        if (values == null || values.length == 0) {
            throw new IllegalArgumentException("Values cannot be null or empty.");
        }
        double total = 0;
        for (N value : values) {
            if (value == null) {
                throw new IllegalArgumentException("Values cannot contain null.");
            }
            total += value.doubleValue();
        }
        return total / values.length;
    }
}

class StatePersister {
    public static void save(Account[] accounts, Path file) throws IOException {
        Path parent = file.toAbsolutePath().getParent();
        if (parent != null) {
            Files.createDirectories(parent);
        }
        try (ObjectOutputStream out = new ObjectOutputStream(Files.newOutputStream(file))) {
            out.writeObject(accounts);
        }
    }

    public static Account[] load(Path file)
            throws IOException, ClassNotFoundException {
        try (ObjectInputStream in = new ObjectInputStream(Files.newInputStream(file))) {
            return (Account[]) in.readObject();
        }
    }
}

class TransactionLog {
    public static synchronized void append(Path file, String line) {
        try {
            Path parent = file.toAbsolutePath().getParent();
            if (parent != null) {
                Files.createDirectories(parent);
            }
            Files.write(
                    file,
                    (line + System.lineSeparator()).getBytes(java.nio.charset.StandardCharsets.UTF_8),
                    StandardOpenOption.CREATE,
                    StandardOpenOption.APPEND);
        } catch (IOException e) {
            throw new UncheckedIOException("Could not write transaction log.", e);
        }
    }
}

class ReportGenerator {
    public static void generate(Path logsFolder, Path reportFile) throws IOException {
        long totalDeposits = 0;
        long totalWithdrawals = 0;
        StringBuilder report = new StringBuilder();
        report.append("MINIBANK END-OF-DAY REPORT\n");
        report.append("==========================\n");

        if (Files.exists(logsFolder)) {
            try (DirectoryStream<Path> stream = Files.newDirectoryStream(logsFolder)) {
                for (Path file : stream) {
                    if (!Files.isRegularFile(file)) {
                        continue;
                    }
                    BasicFileAttributes attributes =
                            Files.readAttributes(file, BasicFileAttributes.class);
                    report.append("\nFile: ").append(file.getFileName()).append("\n");
                    report.append("Size: ").append(attributes.size()).append(" bytes\n");
                    report.append("Last modified: ")
                            .append(attributes.lastModifiedTime()).append("\n");

                    try (BufferedReader reader = Files.newBufferedReader(file)) {
                        String line;
                        while ((line = reader.readLine()) != null) {
                            String[] parts = line.split("\\|");
                            if (parts.length >= 3) {
                                try {
                                    long amount = Long.parseLong(parts[2]);
                                    if ("DEPOSIT".equals(parts[0])) {
                                        totalDeposits += amount;
                                    } else if ("WITHDRAW".equals(parts[0])) {
                                        totalWithdrawals += amount;
                                    }
                                } catch (NumberFormatException ignored) {
                                    // Ignore malformed log entries.
                                }
                            }
                        }
                    }
                }
            }
        } else {
            report.append("\nNo logs folder found.\n");
        }

        report.append("\nTOTAL DEPOSITS: ").append(totalDeposits).append("\n");
        report.append("TOTAL WITHDRAWALS: ").append(totalWithdrawals).append("\n");
        report.append("NET FLOW: ").append(totalDeposits - totalWithdrawals).append("\n");

        Path parent = reportFile.toAbsolutePath().getParent();
        if (parent != null) {
            Files.createDirectories(parent);
        }
        Files.writeString(reportFile, report.toString(),
                StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING);
    }
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Audit {}

class ReflectiveAuditLogger {
    private final Path file;

    ReflectiveAuditLogger(Path file) {
        this.file = file;
    }

    public void record(String methodName) {
        TransactionLog.append(file, "AUDIT|" + java.time.LocalDateTime.now()
                + "|" + methodName);
    }

    public void printAuditedMethods(Class<?> serviceClass) {
        System.out.println("Audited service methods:");
        for (java.lang.reflect.Method method : serviceClass.getDeclaredMethods()) {
            if (method.isAnnotationPresent(Audit.class)) {
                System.out.println("- " + method.getName());
            }
        }
    }
}

class BankService {
    private final Repository<Account> repository;
    private final TransactionProcessor processor;
    private final AccountStore accountStore = new AccountStore();
    private final Path transactionLog;
    private final ReflectiveAuditLogger auditLogger;
    private final List<String> history = Collections.synchronizedList(new ArrayList<>());

    BankService(Repository<Account> repository, TransactionProcessor processor,
                Path transactionLog) {
        this.repository = repository;
        this.processor = processor;
        this.transactionLog = transactionLog;
        this.auditLogger = new ReflectiveAuditLogger(transactionLog);
        for (Account account : repository.findAll()) {
            processor.addAccount(account);
            accountStore.add(account);
        }
    }

    private void audit(String method) {
        auditLogger.record(method);
    }

    @Audit
    public Account openAccount(String ownerName, String accountType, long openingBalance)
            throws BankException {
        audit("openAccount");
        if (!Validator.isValidName(ownerName)) {
            throw new BankException("Owner name must contain 1-30 characters.");
        }
        if (openingBalance < 0) {
            throw new InvalidAmountException("Opening balance cannot be negative.");
        }
        Account account;
        switch (accountType.trim().toUpperCase()) {
            case "1", "SAVINGS" -> account = new SavingsAccount(ownerName.trim(),
                    openingBalance, 0);
            case "2", "CURRENT" -> account = new CurrentAccount(ownerName.trim(),
                    openingBalance, 0);
            case "3", "FIXED", "FIXED DEPOSIT" -> account = new FixedDepositAccount(
                    ownerName.trim(), openingBalance);
            default -> throw new BankException("Choose Savings, Current, or Fixed Deposit.");
        }
        repository.save(account);
        processor.addAccount(account);
        accountStore.add(account);
        return account;
    }

    private Account requireAccount(String id) throws AccountNotFoundException {
        Account account = repository.findById(id);
        if (account == null) throw new AccountNotFoundException("Account not found: " + id);
        return account;
    }

    @Audit
    public void deposit(String accountNumber, long amount) throws BankException {
        audit("deposit");
        Account account = requireAccount(accountNumber);
        if (!Validator.isValidAmount(amount)) {
            throw new InvalidAmountException("Deposit amount must be greater than zero.");
        }
        account.deposit(amount);
        record("DEPOSIT", accountNumber, amount);
    }

    @Audit
    public void withdraw(String accountNumber, long amount) throws BankException {
        audit("withdraw");
        Account account = requireAccount(accountNumber);
        if (!Validator.isValidAmount(amount)) {
            throw new InvalidAmountException("Withdrawal amount must be greater than zero.");
        }
        account.withdraw(amount);
        record("WITHDRAW", accountNumber, amount);
    }

    @Audit
    public void transfer(String fromAccountNumber, String toAccountNumber, long amount)
            throws BankException {
        audit("transfer");
        if (fromAccountNumber.equals(toAccountNumber)) {
            throw new BankException("Source and destination accounts must be different.");
        }
        Account from = requireAccount(fromAccountNumber);
        Account to = requireAccount(toAccountNumber);

        // Lock both accounts in a stable order to avoid lock-order deadlocks.
        Account first = from.getAccountNumber().compareTo(to.getAccountNumber()) < 0 ? from : to;
        Account second = first == from ? to : from;
        synchronized (first) {
            synchronized (second) {
                from.withdraw(amount);
                try {
                    to.deposit(amount);
                } catch (BankException e) {
                    from.deposit(amount);
                    throw e;
                }
            }
        }
        record("WITHDRAW", fromAccountNumber, amount);
        record("DEPOSIT", toAccountNumber, amount);
        record("TRANSFER", fromAccountNumber + "->" + toAccountNumber, amount);
    }

    @Audit
    public String statement(String accountNumber) throws BankException {
        audit("statement");
        return StatementFormatter.buildStatement(requireAccount(accountNumber));
    }

    @Audit
    public List<Account> listAccounts() {
        audit("listAccounts");
        ArrayList<Account> accounts = repository.findAll();
        accounts.sort(Comparator.comparing(Account::getAccountNumber));
        return accounts;
    }

    public List<String> getHistory() {
        synchronized (history) {
            return new ArrayList<>(history);
        }
    }

    private void record(String type, String account, long amount) {
        String line = type + "|" + account + "|" + amount;
        TransactionLog.append(transactionLog, line);
        history.add(line);
        accountStore.addTransaction(line);
    }

    @Audit
    public void processBatch(List<Command> commands) throws BankException {
        audit("processBatch");
        if (commands == null) throw new BankException("Transaction batch cannot be null.");
        CountDownLatch finished = new CountDownLatch(commands.size());
        List<String> errors = Collections.synchronizedList(new ArrayList<>());

        for (Command command : commands) {
            requireAccount(command.accountNumber());
            if (command.amount() <= 0) {
                throw new InvalidAmountException("Batch amounts must be greater than zero.");
            }
        }

        for (Command command : commands) {
            processor.submit(() -> {
                try {
                    if (command.type() == TransactionType.DEPOSIT) {
                        deposit(command.accountNumber(), command.amount());
                    } else if (command.type() == TransactionType.WITHDRAW) {
                        withdraw(command.accountNumber(), command.amount());
                    } else {
                        errors.add("Batch supports DEPOSIT and WITHDRAW only: " + command.type());
                    }
                } catch (BankException e) {
                    errors.add(command.type() + " for " + command.accountNumber()
                            + ": " + e.getMessage());
                } finally {
                    finished.countDown();
                }
            });
        }

        try {
            if (!finished.await(30, TimeUnit.SECONDS)) {
                throw new BankException("Batch timed out before all transactions completed.");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new BankException("Batch processing was interrupted.");
        }
        if (!errors.isEmpty())
            throw new BankException("Batch completed with issues: " + String.join("; ", errors));
    }

    public void shutdown() throws InterruptedException {
        processor.stop();
    }
}

public class MiniBank {
    private static final Path LOGS_FOLDER = Paths.get("logs");
    private static final Path TRANSACTION_LOG = LOGS_FOLDER.resolve("transactions.log");
    private static final Path ACCOUNT_FILE = Paths.get("minibank-accounts.dat");
    private static final Path REPORT_FILE = Paths.get("end-of-day-report.txt");

    private static void recordTransaction(AccountStore store, Path file, String line) {
        TransactionLog.append(file, line);
        store.addTransaction(line);
    }

    public static void runMultithreadingTest() {
        int threadCount = 10;
        int times = 1000;
        long amount = 10;
        long expected = (long) threadCount * times * amount;

        System.out.println("\nMULTITHREADING TEST");
        System.out.println("\nWITHOUT SYNCHRONIZATION");

        Account unsafeAccount = new SavingsAccount("Unsafe User", 0, 0);
        CountDownLatch unsafeLatch = new CountDownLatch(1);
        Thread[] unsafeThreads = new Thread[threadCount];

        for (int i = 0; i < threadCount; i++) {
            unsafeThreads[i] = new Thread(
                    new AccountWorker(unsafeAccount, times, amount, false, unsafeLatch),
                    "Unsafe-" + (i + 1));
            unsafeThreads[i].start();
        }

        unsafeLatch.countDown();
        try {
            for (Thread thread : unsafeThreads) {
                thread.join();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.out.println("Main thread interrupted.");
            return;
        }

        System.out.println("Expected balance: " + expected);
        System.out.println("Final balance: " + unsafeAccount.getBalance());

        System.out.println("\nWITH SYNCHRONIZATION");
        Account safeAccount = new SavingsAccount("Safe User", 0, 0);
        CountDownLatch safeLatch = new CountDownLatch(1);
        Thread[] safeThreads = new Thread[threadCount];

        for (int i = 0; i < threadCount; i++) {
            safeThreads[i] = new Thread(
                    new AccountWorker(safeAccount, times, amount, true, safeLatch),
                    "Safe-" + (i + 1));
            safeThreads[i].start();
        }

        safeLatch.countDown();
        try {
            for (Thread thread : safeThreads) {
                thread.join();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.out.println("Main thread interrupted.");
            return;
        }

        System.out.println("Expected balance: " + expected);
        System.out.println("Final balance: " + safeAccount.getBalance());
        System.out.println("MULTITHREADING TEST COMPLETED.");
    }

    public static void runPartBProjectWork() {
        System.out.println("\n========== PART B: TRANSACTION ENGINE ==========");

        Account poolAccount = new SavingsAccount("Pool User", 10000, 0);
        TransactionProcessor processor = new TransactionProcessor();
        processor.addAccount(poolAccount);

        System.out.println("\n--- 1. Fixed thread pool: deposit and withdrawal tasks ---");
        for (int i = 0; i < 6; i++) {
            processor.submit(new BankTransactionTask(
                    poolAccount, TransactionType.DEPOSIT, 100));
            processor.submit(new BankTransactionTask(
                    poolAccount, TransactionType.WITHDRAW, 50));
        }
        try {
            processor.stop();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.out.println("Interrupted while stopping transaction processor.");
            return;
        }
        System.out.println("All thread-pool tasks completed. Final balance: "
                + poolAccount.getBalance());

        System.out.println("\n--- 2. Producer-consumer transaction buffer ---");
        Account bufferAccount = new SavingsAccount("Buffer User", 5000, 0);
        TransactionBuffer buffer = new TransactionBuffer(3);
        Runnable poisonPill = () -> {};

        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 8; i++) {
                    TransactionType type = (i % 2 == 0)
                            ? TransactionType.WITHDRAW
                            : TransactionType.DEPOSIT;
                    long amount = 100;
                    System.out.println("Producer queued transaction " + i
                            + ": " + type + " " + amount);
                    buffer.put(new BankTransactionTask(bufferAccount, type, amount));
                }
                buffer.put(poisonPill);
                System.out.println("Producer finished.");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                System.out.println("Producer interrupted.");
            }
        }, "Transaction-Producer");

        Thread consumer = new Thread(() -> {
            try {
                while (true) {
                    Runnable task = buffer.take();
                    if (task == poisonPill) {
                        break;
                    }
                    task.run();
                }
                System.out.println("Consumer finished.");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                System.out.println("Consumer interrupted.");
            }
        }, "Transaction-Consumer");

        producer.start();
        consumer.start();
        try {
            producer.join();
            consumer.join();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.out.println("Main thread interrupted while waiting for buffer.");
            return;
        }
        System.out.println("Producer-consumer final balance: "
                + bufferAccount.getBalance());

        try {
            TransferLockDemo.demonstrateDeadlock();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.out.println("Deadlock demonstration interrupted.");
        }
        System.out.println("========== PART B COMPLETED ==========");
    }

    private static void runValidationAndParserDemo() {
        System.out.println("Valid mobile 9876543210: " + Validator.isValidMobile("9876543210"));
        System.out.println("Invalid mobile 12345: " + Validator.isValidMobile("12345"));
        System.out.println("Valid email student@example.com: "
                + Validator.isValidEmail("student@example.com"));
        System.out.println("Invalid email student@: " + Validator.isValidEmail("student@"));
        System.out.println("Valid PAN ABCDE1234F: " + Validator.isValidPan("ABCDE1234F"));
        System.out.println("Invalid PAN abc123: " + Validator.isValidPan("abc123"));
        System.out.println("Valid IFSC SBIN0001234: " + Validator.isValidIfsc("SBIN0001234"));
        System.out.println("Invalid IFSC XYZ: " + Validator.isValidIfsc("XYZ"));
        Command command = CommandParser.parse("DEPOSIT AC0001 500");
        System.out.println("Parsed command: " + command.type() + ", "
                + command.accountNumber() + ", " + command.amount());
        System.out.println("Annotation validation for a negative-balance account:");
        for (String error : AnnotationValidator.validate(
                new SavingsAccount("Invalid Example", -10, 0))) {
            System.out.println("- " + error);
        }
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        Repository<Account> repository = new Repository<>();

        try {
            if (Files.exists(ACCOUNT_FILE)) {
                Account[] saved = StatePersister.load(ACCOUNT_FILE);
                for (Account account : saved)
                    Account.ensureCounterAbove(account.getAccountNumber());
                repository.saveAll(saved);
                System.out.println("Loaded " + saved.length + " saved account(s).");
            } else {
                repository.save(new SavingsAccount("Pruthvi", 10000, 5000));
                repository.save(new CurrentAccount("Rahul", 5000, 3000));
                repository.save(new FixedDepositAccount("Amit", 20000));
                System.out.println("No saved data found. Created three sample accounts.");
            }
        } catch (IOException | ClassNotFoundException e) {
            System.out.println("Could not load saved accounts: " + e.getMessage());
            repository.save(new SavingsAccount("Pruthvi", 10000, 5000));
        }

        TransactionProcessor processor = new TransactionProcessor();
        BankService service = new BankService(repository, processor, TRANSACTION_LOG);

        boolean running = true;
        BankInfo bankInfo = new BankInfo("MiniBank", "ABC Bank, Surat Branch");
        System.out.println("\n================================");
        System.out.println(bankInfo.name() + " - " + bankInfo.branch());
        System.out.println("================================");

        runValidationAndParserDemo();

        while (running) {
            System.out.println("\n------------- MINIBANK MENU ------------- ");
            System.out.println("1. Open account");
            System.out.println("2. Deposit");
            System.out.println("3. Withdraw");
            System.out.println("4. Transfer");
            System.out.println("5. Account statement");
            System.out.println("6. List accounts");
            System.out.println("7. Process transaction batch");
            System.out.println("8. Generate end-of-day report");
            System.out.println("9. Exit and save");
            System.out.print("Enter choice: ");

            String rawChoice = sc.nextLine().trim();
            int choice;
            try {
                choice = Integer.parseInt(rawChoice);
            } catch (NumberFormatException e) {
                System.out.println("Enter a number from 1 to 9.");
                continue;
            }

            MenuOption selectedOption = switch (choice) {
                case 1 -> MenuOption.OPEN_ACCOUNT;
                case 2 -> MenuOption.DEPOSIT;
                case 3 -> MenuOption.WITHDRAW;
                case 4 -> MenuOption.TRANSFER;
                case 5 -> MenuOption.STATEMENT;
                case 6 -> MenuOption.LIST_ACCOUNTS;
                case 7 -> MenuOption.BATCH_PROCESSING;
                case 8 -> MenuOption.REPORT;
                case 9 -> MenuOption.EXIT;
                default -> null;
            };

            if (selectedOption == null) {
                System.out.println("Invalid choice. Enter 1-9.");
                continue;
            }

            try {
                switch (selectedOption) {
                    case OPEN_ACCOUNT -> {
                        System.out.print("Owner name: ");
                        String owner = sc.nextLine().trim();
                        System.out.println("Account type: 1 Savings, 2 Current, 3 Fixed Deposit");
                        System.out.print("Choose type: ");
                        String type = sc.nextLine().trim();
                        System.out.print("Opening balance: ");
                        long amount = Long.parseLong(sc.nextLine().trim());
                        Account account = service.openAccount(owner, type, amount);
                        System.out.println("Account created: " + account);
                    }
                    case DEPOSIT -> {
                        System.out.print("Account number: ");
                        String id = sc.nextLine().trim();
                        System.out.print("Deposit amount: ");
                        long amount = Long.parseLong(sc.nextLine().trim());
                        service.deposit(id, amount);
                        System.out.println("Deposit successful.");
                    }
                    case WITHDRAW -> {
                        System.out.print("Account number: ");
                        String id = sc.nextLine().trim();
                        System.out.print("Withdrawal amount: ");
                        long amount = Long.parseLong(sc.nextLine().trim());
                        service.withdraw(id, amount);
                        System.out.println("Withdrawal successful.");
                    }
                    case TRANSFER -> {
                        System.out.print("From account: ");
                        String from = sc.nextLine().trim();
                        System.out.print("To account: ");
                        String to = sc.nextLine().trim();
                        System.out.print("Transfer amount: ");
                        long amount = Long.parseLong(sc.nextLine().trim());
                        service.transfer(from, to, amount);
                        System.out.println("Transfer successful.");
                    }
                    case STATEMENT -> {
                        System.out.print("Account number: ");
                        System.out.println(service.statement(sc.nextLine().trim()));
                    }
                    case LIST_ACCOUNTS -> {
                        List<Account> all = service.listAccounts();
                        if (all.isEmpty()) System.out.println("No accounts available.");
                        else for (Account account : all) System.out.println(account);
                    }
                    case BATCH_PROCESSING -> {
                        System.out.print("Number of transactions in batch: ");
                        int count = Integer.parseInt(sc.nextLine().trim());
                        if (count < 1 || count > 1000)
                            throw new IllegalArgumentException(
                                    "Choose between 1 and 1000 transactions.");
                        List<Command> commands = new ArrayList<>();
                        for (int i = 0; i < count; i++) {
                            System.out.println("Transaction " + (i + 1) + ": 1 Deposit, 2 Withdraw");
                            int type = Integer.parseInt(sc.nextLine().trim());
                            TransactionType transactionType = type == 1
                                    ? TransactionType.DEPOSIT
                                    : type == 2 ? TransactionType.WITHDRAW : null;
                            if (transactionType == null)
                                throw new IllegalArgumentException(
                                        "Transaction type must be 1 or 2.");
                            System.out.print("Account number: ");
                            String id = sc.nextLine().trim();
                            System.out.print("Amount: ");
                            long amount = Long.parseLong(sc.nextLine().trim());
                            commands.add(new Command(transactionType, id, amount));
                        }
                        service.processBatch(commands);
                        System.out.println("Batch processing completed successfully.");
                    }
                    case REPORT -> {
                        ReportGenerator.generate(LOGS_FOLDER, REPORT_FILE);
                        System.out.println(Files.readString(REPORT_FILE));
                        System.out.println("Report saved to " + REPORT_FILE.toAbsolutePath());
                    }
                    case EXIT -> running = false;
                }
            } catch (NumberFormatException e) {
                System.out.println("Invalid number. Please try again.");
            } catch (BankException | IOException e) {
                System.out.println("Operation failed: " + e.getMessage());
            } catch (IllegalArgumentException e) {
                System.out.println("Invalid input: " + e.getMessage());
            }
        }

        try {
            service.shutdown();
            List<Account> all = service.listAccounts();
            StatePersister.save(all.toArray(new Account[0]), ACCOUNT_FILE);
            System.out.println("Accounts saved to " + ACCOUNT_FILE.toAbsolutePath());
            System.out.println("Transaction history:");
            for (String line : service.getHistory()) System.out.println(line);
            System.out.println("Thank you for using MiniBank.");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.out.println("Interrupted while stopping transaction processor.");
        } catch (IOException e) {
            System.out.println("Could not save accounts: " + e.getMessage());
        }
        sc.close();
    }
}
