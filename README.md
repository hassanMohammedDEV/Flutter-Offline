# Flutter Offline-First Example with Dio & Hive Cache

هذا المشروع مثال كامل يوضح كيفية جلب البيانات من API وحفظها محليًا باستخدام **HiveCacheStore**، بحيث يمكن الوصول للبيانات حتى في وضع **عدم الاتصال بالإنترنت**، ويستمر حفظها بعد إغلاق التطبيق.

---

## 📦 المتطلبات

* Flutter >= 3.0
* Dart >= 3.0
* Packages:

```yaml
dependencies:
  flutter:
    sdk: flutter
  dio: ^5.0.0
  dio_cache_interceptor: ^4.0.0
  dio_cache_interceptor_hive_store: ^3.0.0
  hive: ^2.2.3
  path_provider: ^2.1.2
```

---

## ⚙️ إعداد Dio مع Cache

```dart
import 'package:dio/dio.dart';
import 'package:dio_cache_interceptor/dio_cache_interceptor.dart';
import 'package:dio_cache_interceptor_hive_store/dio_cache_interceptor_hive_store.dart';
import 'package:path_provider/path_provider.dart';

class DioClient {
  late Dio dio;

  DioClient._create();

  static Future<DioClient> create() async {
    final client = DioClient._create();
    final dir = await getApplicationDocumentsDirectory();

    final store = HiveCacheStore(dir.path, hiveBoxName: 'my_app_cache');

    final cacheOptions = CacheOptions(
      store: store,
      policy: CachePolicy.cacheFirst, // استخدام الكاش أولاً
      maxStale: const Duration(days: 30),
      hitCacheOnErrorExcept: [401, 403],
    );

    client.dio = Dio(BaseOptions(baseUrl: 'https://example.com/api'))
      ..interceptors.add(DioCacheInterceptor(options: cacheOptions));

    return client;
  }
}
```

---

## 🗂️ نموذج البيانات

```dart
class Category {
  final int id;
  final String name;
  Category({required this.id, required this.name});
  factory Category.fromJson(Map<String, dynamic> json) =>
      Category(id: json['id'], name: json['name']);
}

class Currency {
  final int id;
  final String code;
  Currency({required this.id, required this.code});
  factory Currency.fromJson(Map<String, dynamic> json) =>
      Currency(id: json['id'], code: json['code']);
}

class Product {
  final int id;
  final String name;
  final int categoryId;
  final int currencyId;
  Product({
    required this.id,
    required this.name,
    required this.categoryId,
    required this.currencyId,
  });
  factory Product.fromJson(Map<String, dynamic> json) => Product(
        id: json['id'],
        name: json['name'],
        categoryId: json['categoryId'],
        currencyId: json['currencyId'],
      );
}
```

---

## 🌐 جلب البيانات مع Cache

```dart
class ApiService {
  final DioClient dioClient;
  ApiService(this.dioClient);

  Future<List<Category>> fetchCategories() async {
    final res = await dioClient.dio.get(
      '/categories',
      options: CacheOptions(policy: CachePolicy.cacheFirst).toOptions(),
    );
    return (res.data as List).map((e) => Category.fromJson(e)).toList();
  }

  Future<List<Currency>> fetchCurrencies() async {
    final res = await dioClient.dio.get(
      '/currencies',
      options: CacheOptions(policy: CachePolicy.cacheFirst).toOptions(),
    );
    return (res.data as List).map((e) => Currency.fromJson(e)).toList();
  }

  Future<List<Product>> fetchProducts() async {
    final res = await dioClient.dio.get(
      '/products',
      options: CacheOptions(policy: CachePolicy.cacheFirst).toOptions(),
    );
    return (res.data as List).map((e) => Product.fromJson(e)).toList();
  }
}
```

---

## 🖥️ استخدام البيانات في Flutter

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final dioClient = await DioClient.create();
  runApp(MyApp(dioClient: dioClient));
}

class MyApp extends StatelessWidget {
  final DioClient dioClient;
  const MyApp({super.key, required this.dioClient});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: HomeScreen(apiService: ApiService(dioClient)),
    );
  }
}

class HomeScreen extends StatefulWidget {
  final ApiService apiService;
  const HomeScreen({super.key, required this.apiService});

  @override
  State<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  late Future<List<Product>> productsFuture;
  late Future<List<Category>> categoriesFuture;
  late Future<List<Currency>> currenciesFuture;

  @override
  void initState() {
    super.initState();
    productsFuture = widget.apiService.fetchProducts();
    categoriesFuture = widget.apiService.fetchCategories();
    currenciesFuture = widget.apiService.fetchCurrencies();
  }

  String getCategoryName(int id, List<Category> categories) {
    return categories.firstWhere((c) => c.id == id, orElse: () => Category(id: 0, name: 'Unknown')).name;
  }

  String getCurrencyCode(int id, List<Currency> currencies) {
    return currencies.firstWhere((c) => c.id == id, orElse: () => Currency(id: 0, code: '---')).code;
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Products Offline-First')),
      body: FutureBuilder(
        future: Future.wait([productsFuture, categoriesFuture, currenciesFuture]),
        builder: (context, snapshot) {
          if (!snapshot.hasData) return Center(child: CircularProgressIndicator());

          final products = snapshot.data![0] as List<Product>;
          final categories = snapshot.data![1] as List<Category>;
          final currencies = snapshot.data![2] as List<Currency>;

          return ListView.builder(
            itemCount: products.length,
            itemBuilder: (context, i) {
              final p = products[i];
              return ListTile(
                title: Text(p.name),
                subtitle: Text('${getCategoryName(p.categoryId, categories)} | ${getCurrencyCode(p.currencyId, currencies)}'),
              );
            },
          );
        },
      ),
    );
  }
}
```

---

## ✅ المميزات

* Offline-First: عرض البيانات من الكاش أولاً إذا لم يتوفر الإنترنت.
* Cache دائم: البيانات محفوظة على الجهاز حتى بعد إغلاق التطبيق.
* تحديث تلقائي: إذا كان الإنترنت متاحًا، يتم جلب البيانات الجديدة وتحديث الكاش.
* سهولة الاستخدام: يمكن إضافة أي Endpoint آخر بنفس الطريقة.
