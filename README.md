import 'package:flutter/material.dart';
import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Gerenciador de Planetas',
      theme: ThemeData(primarySwatch: Colors.blue),
      home: PlanetListScreen(),
    );
  }
}

class DatabaseHelper {
  static final DatabaseHelper instance = DatabaseHelper._init();
  static Database? _database;

  DatabaseHelper._init();

  Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDB('planets.db');
    return _database!;
  }

  Future<Database> _initDB(String filePath) async {
    final dbPath = await getDatabasesPath();
    final path = join(dbPath, filePath);

    return await openDatabase(
      path,
      version: 1,
      onCreate: (db, version) async {
        await db.execute('''
          CREATE TABLE planets (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            distance REAL NOT NULL,
            size INTEGER NOT NULL,
            nickname TEXT
          )
        ''');
      },
    );
  }

  Future<int> insertPlanet(Map<String, dynamic> planet) async {
    final db = await instance.database;
    return await db.insert('planets', planet);
  }

  Future<List<Map<String, dynamic>>> getPlanets() async {
    final db = await instance.database;
    return await db.query('planets');
  }

  Future<int> updatePlanet(Map<String, dynamic> planet) async {
    final db = await instance.database;
    return await db.update(
      'planets',
      planet,
      where: 'id = ?',
      whereArgs: [planet['id']],
    );
  }

  Future<int> deletePlanet(int id) async {
    final db = await instance.database;
    return await db.delete(
      'planets',
      where: 'id = ?',
      whereArgs: [id],
    );
  }
}

class PlanetListScreen extends StatefulWidget {
  @override
  _PlanetListScreenState createState() => _PlanetListScreenState();
}

class _PlanetListScreenState extends State<PlanetListScreen> {
  List<Map<String, dynamic>> planets = [];

  @override
  void initState() {
    super.initState();
    _loadPlanets();
  }

  Future<void> _loadPlanets() async {
    final data = await DatabaseHelper.instance.getPlanets();
    setState(() {
      planets = data;
    });
  }

  void _deletePlanet(int id) async {
    await DatabaseHelper.instance.deletePlanet(id);
    _loadPlanets();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Lista de Planetas')),
      body: ListView.builder(
        itemCount: planets.length,
        itemBuilder: (context, index) {
          final planet = planets[index];
          return ListTile(
            title: Text(planet['name']),
            subtitle: Text(planet['nickname'] ?? 'Sem apelido'),
            trailing: IconButton(
              icon: Icon(Icons.delete, color: Colors.red),
              onPressed: () => _deletePlanet(planet['id']),
            ),
            onTap: () => Navigator.push(
              context,
              MaterialPageRoute(
                builder: (context) => PlanetFormScreen(planet: planet),
              ),
            ).then((_) => _loadPlanets()),
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        child: Icon(Icons.add),
        onPressed: () => Navigator.push(
          context,
          MaterialPageRoute(
            builder: (context) => PlanetFormScreen(),
          ),
        ).then((_) => _loadPlanets()),
      ),
    );
  }
}

class PlanetFormScreen extends StatefulWidget {
  final Map<String, dynamic>? planet;
  PlanetFormScreen({this.planet});

  @override
  _PlanetFormScreenState createState() => _PlanetFormScreenState();
}

class _PlanetFormScreenState extends State<PlanetFormScreen> {
  final _formKey = GlobalKey<FormState>();
  String name = '';
  String nickname = '';
  double distance = 0;
  int size = 0;

  @override
  void initState() {
    super.initState();
    if (widget.planet != null) {
      name = widget.planet!['name'];
      nickname = widget.planet!['nickname'] ?? '';
      distance = widget.planet!['distance'];
      size = widget.planet!['size'];
    }
  }

  void _savePlanet() async {
    if (_formKey.currentState!.validate()) {
      _formKey.currentState!.save();
      final planet = {
        'name': name,
        'nickname': nickname,
        'distance': distance,
        'size': size,
      };

      if (widget.planet == null) {
        await DatabaseHelper.instance.insertPlanet(planet);
      } else {
        planet['id'] = widget.planet!['id'];
        await DatabaseHelper.instance.updatePlanet(planet);
      }

      Navigator.pop(context);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(widget.planet == null ? 'Adicionar Planeta' : 'Editar Planeta')),
      body: Padding(
        padding: EdgeInsets.all(16.0),
        child: Form(
          key: _formKey,
          child: Column(
            children: [
              TextFormField(
                initialValue: name,
                decoration: InputDecoration(labelText: 'Nome do Planeta'),
                validator: (value) => value!.isEmpty ? 'Preencha o nome' : null,
                onSaved: (value) => name = value!,
              ),
              TextFormField(
                initialValue: nickname,
                decoration: InputDecoration(labelText: 'Apelido (opcional)'),
                onSaved: (value) => nickname = value!,
              ),
              TextFormField(
                initialValue: distance.toString(),
                decoration: InputDecoration(labelText: 'Distância do Sol (UA)'),
                keyboardType: TextInputType.number,
                onSaved: (value) => distance = double.parse(value!),
              ),
              TextFormField(
                initialValue: size.toString(),
                decoration: InputDecoration(labelText: 'Tamanho (km)'),
                keyboardType: TextInputType.number,
                onSaved: (value) => size = int.parse(value!),
              ),
              SizedBox(height: 20),
              ElevatedButton(onPressed: _savePlanet, child: Text('Salvar')),
            ],
          ),
        ),
      ),
    );
  }
}
