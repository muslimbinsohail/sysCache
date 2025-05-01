# sysCache

// features/admin/data/repositories/product_repository.dart
abstract class ProductRepository {
  Future<List<Product>> getAllProducts();
  Future<Product> getProductById(String id);
  Future<void> createProduct(Product product);
  Future<void> updateProduct(Product product);
  Future<void> deleteProduct(String id);
}

class ProductRepositoryImpl implements ProductRepository {
  final ProductService _service;

  ProductRepositoryImpl(this._service);

  @override
  Future<List<Product>> getAllProducts() async {
    try {
      final response = await _service.getAllProducts();
      return response.map((json) => Product.fromJson(json)).toList();
    } catch (error) {
      throw Exception('Failed to load products');
    }
  }

  @override
  Future<Product> getProductById(String id) async {
    try {
      final response = await _service.getProductById(id);
      return Product.fromJson(response);
    } catch (error) {
      throw Exception('Failed to load product');
    }
  }

  @override
  Future<void> createProduct(Product product) async {
    try {
      await _service.createProduct(product.toJson());
    } catch (error) {
      throw Exception('Failed to create product');
    }
  }

  @override
  Future<void> updateProduct(Product product) async {
    try {
      await _service.updateProduct(product.id, product.toJson());
    } catch (error) {
      throw Exception('Failed to update product');
    }
  }

  @override
  Future<void> deleteProduct(String id) async {
    try {
      await _service.deleteProduct(id);
    } catch (error) {
      throw Exception('Failed to delete product');
    }
  }
}


// features/admin/data/services/product_service.dart
import 'package:http/http.dart' as http;
import 'dart:convert';

class ProductService {
  static const String baseUrl = 'your-api-base-url';
  final http.Client client;

  ProductService({http.Client? client})
      : client = client ?? http.Client();

  Future<List<dynamic>> getAllProducts() async {
    final response = await client.get(Uri.parse('$baseUrl/products'));
    
    if (response.statusCode == 200) {
      return jsonDecode(response.body);
    }
    throw Exception('Failed to load products');
  }

  Future<dynamic> getProductById(String id) async {
    final response = await client.get(Uri.parse('$baseUrl/products/$id'));
    
    if (response.statusCode == 200) {
      return jsonDecode(response.body);
    }
    throw Exception('Failed to load product');
  }

  Future<void> createProduct(Map<String, dynamic> product) async {
    final response = await client.post(
      Uri.parse('$baseUrl/products'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode(product),
    );
    
    if (response.statusCode != 200 && response.statusCode != 201) {
      throw Exception('Failed to create product');
    }
  }

  Future<void> updateProduct(String id, Map<String, dynamic> product) async {
    final response = await client.patch(
      Uri.parse('$baseUrl/products/$id'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode(product),
    );
    
    if (response.statusCode != 200) {
      throw Exception('Failed to update product');
    }
  }

  Future<void> deleteProduct(String id) async {
    final response = await client.delete(Uri.parse('$baseUrl/products/$id'));
    
    if (response.statusCode != 200) {
      throw Exception('Failed to delete product');
    }
  }
}




// features/admin/presentation/screens/product_admin_screen.dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../../domain/entities/product.dart';
import '../providers/product_provider.dart';
import '../../../core/responsive/responsive.dart';

class ProductAdminScreen extends StatelessWidget {
  const ProductAdminScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final responsive = Responsive(context);
    final productProvider = Provider.of<ProductProvider>(context);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Product Management'),
        actions: [
          IconButton(
            icon: const Icon(Icons.add),
            onPressed: () {
              showDialog(
                context: context,
                builder: (_) => AddProductDialog(),
              );
            },
          ),
        ],
      ),
      body: RefreshIndicator(
        onRefresh: () async {
          await productProvider.fetchProducts();
        },
        child: ListView.builder(
          itemCount: productProvider.products.length,
          itemBuilder: (ctx, index) {
            final product = productProvider.products[index];
            return FadeScaleAnimation(
              child: AdminProductCard(
                product: product,
                onDelete: () => productProvider.deleteProduct(product.id),
                onUpdate: () {
                  showDialog(
                    context: context,
                    builder: (_) => UpdateProductDialog(
                      initialProduct: product,
                      onSave: (updatedProduct) =>
                          productProvider.updateProduct(updatedProduct),
                    ),
                  );
                },
              ),
            ),
          },
        ),
      ),
    );
  }
}

class AdminProductCard extends StatelessWidget {
  final Product product;
  final VoidCallback onDelete;
  final VoidCallback onUpdate;

  const AdminProductCard({
    required this.product,
    required this.onDelete,
    required this.onUpdate,
    super.key,
  });

  @override
  Widget build(BuildContext context) {
    final responsive = Responsive(context);

    return Card(
      margin: EdgeInsets.symmetric(
        horizontal: responsive.wp(4),
        vertical: responsive.hp(1),
      ),
      elevation: 2,
      child: Padding(
        padding: EdgeInsets.all(responsive.wp(4)),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Text(
                  product.title,
                  style: TextStyle(
                    fontSize: responsive.wp(4.5),
                    fontWeight: FontWeight.bold,
                  ),
                ),
                Row(
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    IconButton(
                      icon: const Icon(Icons.edit),
                      onPressed: onUpdate,
                    ),
                    IconButton(
                      icon: const Icon(Icons.delete),
                      color: Colors.red,
                      onPressed: onDelete,
                    ),
                  ],
                ),
              ],
            ),
            SizedBox(height: responsive.hp(1)),
            Text(
              '\$${product.price.toStringAsFixed(2)}',
              style: TextStyle(
                fontSize: responsive.wp(4),
                fontWeight: FontWeight.w600,
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class AddProductDialog extends StatefulWidget {
  @override
  State<AddProductDialog> createState() => _AddProductDialogState();
}

class _AddProductDialogState extends State<AddProductDialog> {
  final _formKey = GlobalKey<FormState>();
  String _title = '';
  double _price = 0;
  String _description = '';
  List<String> _images = [];
  List<String> _categories = [];

  void _submitForm(BuildContext context) {
    final productProvider = Provider.of<ProductProvider>(context, listen: false);
    
    if (_formKey.currentState!.validate()) {
      productProvider.addProduct(Product(
        id: DateTime.now().toString(),
        title: _title,
        description: _description,
        price: _price,
        images: _images,
        categories: _categories,
      ));
      
      Navigator.of(context).pop();
    }
  }

  @override
  Widget build(BuildContext context) {
    final responsive = Responsive(context);

    return Dialog(
      child: Container(
        width: responsive.wp(80),
        padding: EdgeInsets.all(responsive.wp(4)),
        child: Form(
          key: _formKey,
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              TextFormField(
                decoration: const InputDecoration(labelText: 'Title'),
                validator: (value) {
                  if (value == null || value.isEmpty) {
                    return 'Please enter a title';
                  }
                  return null;
                },
                onSaved: (value) => _title = value!,
              ),
              TextFormField(
                decoration: const InputDecoration(labelText: 'Price'),
                keyboardType: TextInputType.number,
                validator: (value) {
                  if (value == null || value.isEmpty) {
                    return 'Please enter a price';
                  }
                  final price = double.tryParse(value);
                  if (price == null || price <= 0) {
                    return 'Please enter a valid price';
                  }
                  return null;
                },
                onSaved: (value) => _price = double.parse(value!),
              ),
              // Additional fields for description, images, and categories
              SizedBox(height: responsive.hp(3)),
              Row(
                mainAxisAlignment: MainAxisAlignment.end,
                children: [
                  TextButton(
                    onPressed: () => Navigator.of(context).pop(),
                    child: const Text('Cancel'),
                  ),
                  ElevatedButton(
                    onPressed: () => _submitForm(context),
                    child: const Text('Add Product'),
                  ),
                ],
              ),
            ],
          ),
        ),
      ),
    );
  }
}
