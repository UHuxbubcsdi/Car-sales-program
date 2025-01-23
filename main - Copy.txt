#include <iostream>
#include <vector>
#include <string>
#include <algorithm>
#include <fstream>
#include <ctime>
#include <iomanip>
#include <map>
#include <sstream>
#include <limits>
#include <cstring>
#include <cctype>

// Custom encryption key for basic file encryption
const std::string ENCRYPTION_KEY = "CarDealership2024";

class Feedback {
public:
    int rating;
    std::string comment;
    std::string date;

    Feedback(int r, std::string c, std::string d) : rating(r), comment(c), date(d) {}
};

class Car {
public:
    std::string model;
    double price;
    int year;
    int stock;
    std::vector<Feedback> feedbacks;

    Car(std::string m, double p, int y, int s)
            : model(m), price(p), year(y), stock(s) {}
};

class Sale {
public:
    std::string model;
    std::string customerName;
    int customerAge;
    double totalPrice;
    double discount;
    int quantity;
    std::string date;

    Sale(std::string m, std::string cn, int ca, double tp, double d, int q, std::string dt)
            : model(m), customerName(cn), customerAge(ca), totalPrice(tp),
              discount(d), quantity(q), date(dt) {}
};

// Global variables
std::vector<Car> inventory;
std::vector<Sale> sales;

// Function declarations
void initializeInventory();
void displayMenu();
void viewCarStock();
void buyCar();
void viewSalesData();
void saveSalesData();
void loadSalesData();
std::string getCurrentDate();
void encryptDecrypt(std::string& data);
void clearInputBuffer();
bool validateName(const std::string& name);
bool validateAge(int age);
int getFeedbackRating();
std::string getFeedbackComment();

// Initialize inventory with sample data
void initializeInventory() {
    inventory = {
            Car("BMW M3", 65000.0, 2024, 5),
            Car("Mercedes C300", 55000.0, 2023, 3),
            Car("Audi A4", 45000.0, 2023, 4),
            Car("Tesla Model 3", 52000.0, 2024, 6),
            Car("Porsche 911", 120000.0, 2024, 2),
            Car("Toyota Camry", 35000.0, 2023, 8)
    };
}

// Main menu display function
void displayMenu() {
    std::cout << "\n=== Car Dealership Management System ===\n";
    std::cout << "1. View Cars Stock\n";
    std::cout << "2. Buy Car\n";
    std::cout << "3. View Sales Data and Customer Feedback\n";
    std::cout << "4. Exit\n";
    std::cout << "Enter your choice (1-4): ";
}

// Function to sort and display car stock
void viewCarStock() {
    std::cout << "\n=== Current Car Stock ===\n";

    // Create a copy of inventory for sorting
    std::vector<Car> sortedInventory = inventory;

    // Sort by year in descending order
    std::sort(sortedInventory.begin(), sortedInventory.end(),
              [](const Car& a, const Car& b) {
                  return a.year > b.year;
              });

    // Display header
    std::cout << std::left << std::setw(20) << "Model"
              << std::setw(15) << "Year"
              << std::setw(15) << "Price ($)"
              << "Stock\n";
    std::cout << std::string(60, '-') << "\n";

    // Display car information
    for (const auto& car : sortedInventory) {
        std::cout << std::left << std::setw(20) << car.model
                  << std::setw(15) << car.year
                  << std::setw(15) << std::fixed << std::setprecision(2) << car.price
                  << car.stock << "\n";
    }
}

// Function to handle car purchase
void buyCar() {
    viewCarStock();

    std::string modelChoice;
    std::cout << "\nEnter car model to purchase: ";
    std::getline(std::cin, modelChoice);

    // Find the car in inventory
    auto carIt = std::find_if(inventory.begin(), inventory.end(),
                              [&modelChoice](const Car& car) {
                                  return car.model == modelChoice;
                              });

    if (carIt == inventory.end()) {
        std::cout << "Car model not found!\n";
        return;
    }

    if (carIt->stock <= 0) {
        std::cout << "Sorry, this model is out of stock!\n";
        return;
    }

    // Get customer information with validation
    std::string customerName;
    do {
        std::cout << "Enter customer name: ";
        std::getline(std::cin, customerName);
    } while (!validateName(customerName));

    int customerAge;
    do {
        std::cout << "Enter customer age: ";
        std::cin >> customerAge;
        clearInputBuffer();
    } while (!validateAge(customerAge));

    int quantity;
    do {
        std::cout << "Enter quantity (max " << carIt->stock << "): ";
        std::cin >> quantity;
        clearInputBuffer();
    } while (quantity <= 0 || quantity > carIt->stock);

    // Calculate price and apply discount if applicable
    double totalPrice = carIt->price * quantity;
    double discount = 0.0;

    if (quantity >= 2) {
        discount = totalPrice * 0.05; // 5% discount for multiple purchases
        totalPrice -= discount;
    }

    // Confirm purchase
    std::cout << "\nPurchase Summary:\n";
    std::cout << "Model: " << carIt->model << "\n";
    std::cout << "Quantity: " << quantity << "\n";
    std::cout << "Original Price: $" << carIt->price * quantity << "\n";
    std::cout << "Discount: $" << discount << "\n";
    std::cout << "Final Price: $" << totalPrice << "\n";

    char confirm;
    std::cout << "\nConfirm purchase (y/n)? ";
    std::cin >> confirm;
    clearInputBuffer();

    if (std::tolower(confirm) == 'y') {
        // Update inventory
        carIt->stock -= quantity;

        // Record sale
        sales.emplace_back(carIt->model, customerName, customerAge,
                           totalPrice, discount, quantity, getCurrentDate());

        // Get customer feedback
        std::cout << "\nWould you like to leave feedback? (y/n): ";
        std::cin >> confirm;
        clearInputBuffer();

        if (std::tolower(confirm) == 'y') {
            int rating = getFeedbackRating();
            std::string comment = getFeedbackComment();
            carIt->feedbacks.emplace_back(rating, comment, getCurrentDate());
        }

        // Save updated sales data
        saveSalesData();
        std::cout << "\nPurchase completed successfully!\n";
    } else {
        std::cout << "\nPurchase cancelled.\n";
    }
}

// Function to display sales data and feedback
void viewSalesData() {
    if (sales.empty()) {
        std::cout << "\nNo sales data available.\n";
        return;
    }

    // Calculate total sales by model
    std::map<std::string, std::pair<int, double>> salesByModel;
    for (const auto& sale : sales) {
        auto& data = salesByModel[sale.model];
        data.first += sale.quantity;
        data.second += sale.totalPrice;
    }

    // Convert to vector for sorting
    std::vector<std::pair<std::string, std::pair<int, double>>> sortedSales(
            salesByModel.begin(), salesByModel.end());

    // Sort by total sales amount in descending order

    std::sort(sortedSales.begin(), sortedSales.end(),
              [](const auto& a, const auto& b) {
                  return a.second.second > b.second.second;
              });

    // Display sales summary
    std::cout << "\n=== Sales Summary ===\n";
    std::cout << std::left << std::setw(20) << "Model"
              << std::setw(15) << "Units Sold"
              << "Total Sales ($)\n";
    std::cout << std::string(60, '-') << "\n";

    for (const auto& sale : sortedSales) {
        std::cout << std::left << std::setw(20) << sale.first
                  << std::setw(15) << sale.second.first
                  << std::fixed << std::setprecision(2) << sale.second.second << "\n";
    }

    // Display detailed sales data
    std::cout << "\n=== Detailed Sales Data ===\n";
    for (const auto& sale : sales) {
        std::cout << "\nModel: " << sale.model << "\n";
        std::cout << "Customer: " << sale.customerName << "\n";
        std::cout << "Date: " << sale.date << "\n";
        std::cout << "Quantity: " << sale.quantity << "\n";
        std::cout << "Total Price: $" << sale.totalPrice << "\n";
        if (sale.discount > 0) {
            std::cout << "Discount Applied: $" << sale.discount << "\n";
        }
    }

    // Display feedback for each model
    std::cout << "\n=== Customer Feedback ===\n";
    for (const auto& car : inventory) {
        if (!car.feedbacks.empty()) {
            std::cout << "\nFeedback for " << car.model << ":\n";
            for (const auto& feedback : car.feedbacks) {
                std::cout << "Date: " << feedback.date << "\n";
                std::cout << "Rating: " << std::string(feedback.rating, '*') << "\n";
                std::cout << "Comment: " << feedback.comment << "\n";
                std::cout << "---\n";
            }
        }
    }
}

// Function to save sales data to file with encryption
void saveSalesData() {
    std::ofstream file("sales_data.txt");
    if (!file) {
        std::cout << "Error: Unable to save sales data!\n";
        return;
    }

    std::stringstream ss;
    for (const auto& sale : sales) {
        ss << sale.model << "|"
           << sale.customerName << "|"
           << sale.customerAge << "|"
           << sale.totalPrice << "|"
           << sale.discount << "|"
           << sale.quantity << "|"
           << sale.date << "\n";
    }

    std::string data = ss.str();
    encryptDecrypt(data);
    file << data;
    file.close();
}

// Function to load sales data from file with decryption
void loadSalesData() {
    std::ifstream file("sales_data.txt");
    if (!file) {
        return; // File doesn't exist yet
    }

    std::stringstream ss;
    ss << file.rdbuf();
    std::string data = ss.str();
    file.close();

    if (data.empty()) {
        return;
    }

    encryptDecrypt(data);
    std::stringstream decrypted(data);
    std::string line;

    while (std::getline(decrypted, line)) {
        std::stringstream lineStream(line);
        std::string model, customerName, date;
        int customerAge, quantity;
        double totalPrice, discount;

        std::getline(lineStream, model, '|');
        std::getline(lineStream, customerName, '|');
        lineStream >> customerAge;
        lineStream.ignore();
        lineStream >> totalPrice;
        lineStream.ignore();
        lineStream >> discount;
        lineStream.ignore();
        lineStream >> quantity;
        lineStream.ignore();
        std::getline(lineStream, date);

        sales.emplace_back(model, customerName, customerAge,
                           totalPrice, discount, quantity, date);
    }
}

// Simple XOR encryption/decryption
void encryptDecrypt(std::string& data) {
    for (size_t i = 0; i < data.length(); ++i) {
        data[i] ^= ENCRYPTION_KEY[i % ENCRYPTION_KEY.length()];
    }
}

// Utility function to get current date
std::string getCurrentDate() {
    time_t now = time(nullptr);
    char buffer[11];
    strftime(buffer, sizeof(buffer), "%Y-%m-%d", localtime(&now));
    return std::string(buffer);
}

// Input validation functions
void clearInputBuffer() {
    std::cin.clear();
    std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
}
// Name validation
bool validateName(const std::string& name) {
    if (name.empty()) {
        std::cout << "Name cannot be empty!\n";
        return false;
    }
    if (name.length() < 2) {
        std::cout << "Name must be at least 2 characters long!\n";
        return false;
    }
    for (char c : name) {
        if (!std::isalpha(c) && !std::isspace(c)) {
            std::cout << "Name can only contain letters and spaces!\n";
            return false;
        }
    }
    return true;
}
// Age validation
bool validateAge(int age) {
    if (std::cin.fail()) {
        std::cout << "Please enter a valid number!\n";
        clearInputBuffer();
        return false;
    }
    if (age < 18 || age > 120) {
        std::cout << "Age must be between 18 and 120!\n";
        return false;
    }
    return true;
}

int getFeedbackRating() {
    int rating;
    do {
        std::cout << "Enter rating (1-5): ";
        std::cin >> rating;
        clearInputBuffer();
        if (rating < 1 || rating > 5) {
            std::cout << "Rating must be between 1 and 5!\n";
        }
    } while (rating < 1 || rating > 5);
    return rating;
}

std::string getFeedbackComment() {
    std::string comment;
    std::cout << "Enter your comment: ";
    std::getline(std::cin, comment);
    return comment;
}

int main() {
    // Initialize car inventory
    initializeInventory();

    // Load previous sales data
    loadSalesData();

    int choice;
    do {
        displayMenu();
        std::cin >> choice;
        clearInputBuffer();

        switch (choice) {
            case 1:
                viewCarStock();
                break;
            case 2:
                buyCar();
                break;
            case 3:
                viewSalesData();
                break;
            case 4:
                std::cout << "\nThank you for using the Car Dealership Management System!\n";
                break;
            default:
                std::cout << "\nInvalid choice! Please try again.\n";
        }
    } while (choice != 4);

    return 0;
}