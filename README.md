import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(const ChatApp());
}

class ChatApp extends StatelessWidget {
  const ChatApp({super.key});
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Do‘stlar Chat',
      home: FirebaseAuth.instance.currentUser == null
          ? const LoginPage()
          : const ChatPage(),
    );
  }
}

class LoginPage extends StatefulWidget {
  const LoginPage({super.key});
  @override
  State<LoginPage> createState() => _LoginPageState();
}

class _LoginPageState extends State<LoginPage> {
  final emailController = TextEditingController();
  final passController = TextEditingController();

  Future<void> registerOrLogin() async {
    try {
      // ro‘yxatdan o‘tish yoki login
      await FirebaseAuth.instance.signInWithEmailAndPassword(
        email: emailController.text,
        password: passController.text,
      );
    } catch (e) {
      await FirebaseAuth.instance.createUserWithEmailAndPassword(
        email: emailController.text,
        password: passController.text,
      );
    }
    if (mounted) {
      Navigator.pushReplacement(
          context, MaterialPageRoute(builder: (_) => const ChatPage()));
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text("Kirish")),
      body: Padding(
        padding: const EdgeInsets.all(20),
        child: Column(
          children: [
            TextField(controller: emailController, decoration: const InputDecoration(labelText: "Email")),
            TextField(controller: passController, decoration: const InputDecoration(labelText: "Parol"), obscureText: true),
            const SizedBox(height: 20),
            ElevatedButton(onPressed: registerOrLogin, child: const Text("Kirish"))
          ],
        ),
      ),
    );
  }
}

class ChatPage extends StatefulWidget {
  const ChatPage({super.key});
  @override
  State<ChatPage> createState() => _ChatPageState();
}

class _ChatPageState extends State<ChatPage> {
  final msgController = TextEditingController();
  final user = FirebaseAuth.instance.currentUser!;
  final adminEmail = "admin@example.com"; // SENING ADMIN EMAILING

  Future<void> sendMessage() async {
    // foydalanuvchi limiti (10 ta)
    final users = await FirebaseFirestore.instance.collection("users").get();
    if (!users.docs.any((d) => d.id == user.uid)) {
      if (users.size >= 10) {
        ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text("Limit 10 foydalanuvchi!")));
        return;
      }
      await FirebaseFirestore.instance.collection("users").doc(user.uid).set({
        "email": user.email,
      });
    }

    if (msgController.text.trim().isEmpty) return;
    await FirebaseFirestore.instance.collection("messages").add({
      "text": msgController.text,
      "user": user.email,
      "time": FieldValue.serverTimestamp(),
    });
    msgController.clear();
  }

  Future<void> deleteMessage(String id) async {
    await FirebaseFirestore.instance.collection("messages").doc(id).delete();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text("Do‘stlar chat xonasi"),
        actions: [
          IconButton(
              onPressed: () async {
                await FirebaseAuth.instance.signOut();
                if (mounted) {
                  Navigator.pushReplacement(context, MaterialPageRoute(builder: (_) => const LoginPage()));
                }
              },
              icon: const Icon(Icons.exit_to_app))
        ],
      ),
      body: Column(
        children: [
          Expanded(
            child: StreamBuilder(
              stream: FirebaseFirestore.instance.collection("messages").orderBy("time").snapshots(),
              builder: (context, AsyncSnapshot<QuerySnapshot> snapshot) {
                if (!snapshot.hasData) return const Center(child: CircularProgressIndicator());
                return ListView(
                  children: snapshot.data!.docs.map((doc) {
                    final data = doc.data() as Map<String, dynamic>;
                    final isAdmin = user.email == adminEmail;
                    return ListTile(
                      title: Text(data["text"] ?? ""),
                      subtitle: Text(data["user"] ?? ""),
                      trailing: isAdmin
                          ? IconButton(
                              icon: const Icon(Icons.delete),
                              onPressed: () => deleteMessage(doc.id),
                            )
                          : null,
                    );
                  }).toList(),
                );
              },
            ),
          ),
          Row(
            children: [
              Expanded(
                child: TextField(controller: msgController, decoration: const InputDecoration(hintText: "Xabar yozing...")),
              ),
              IconButton(onPressed: sendMessage, icon: const Icon(Icons.send))
            ],
          )
        ],
      ),
    );
  }
}
