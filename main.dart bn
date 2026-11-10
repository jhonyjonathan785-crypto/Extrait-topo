// Extrait topographique - fichier unique (voir LISEZMOI)
import 'dart:io';
import 'dart:math';
import 'dart:typed_data';
import 'package:file_picker/file_picker.dart';
import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;
import 'package:image/image.dart' as img;
import 'package:path_provider/path_provider.dart';
import 'package:pdf/pdf.dart';
import 'package:printing/printing.dart';


class Borne {
  final String n;
  final double x, y;
  double g = 0, d = 0; // angle (gr) au sommet, distance vers la borne suivante
  Borne(this.n, this.x, this.y);
}

class Calcul {
  final double surface, perimetre;
  Calcul(this.surface, this.perimetre);
  String get contenance {
    final t = surface.round();
    final a = ((t % 10000) ~/ 100).toString().padLeft(2, '0');
    final c = (t % 100).toString().padLeft(2, '0');
    return '${t ~/ 10000} ha $a a $c ca';
  }
}

double _gis(Borne a, Borne b) =>
    ((atan2(b.x - a.x, b.y - a.y) * 200 / pi) % 400 + 400) % 400;

Calcul calculer(List<Borne> p) {
  final n = p.length;
  double s = 0, per = 0;
  for (var i = 0; i < n; i++) {
    final a = p[i], b = p[(i + 1) % n], c = p[(i + n - 1) % n];
    s += a.x * b.y - b.x * a.y;
    a.d = sqrt((b.x - a.x) * (b.x - a.x) + (b.y - a.y) * (b.y - a.y));
    per += a.d;
    a.g = ((_gis(a, c) - _gis(a, b)) % 400 + 400) % 400;
  }
  return Calcul(s.abs() / 2, per);
}

List<Borne> lireBornes(String t) {
  final r = <Borne>[];
  for (final l in t.split('\n')) {
    final s = l.trim();
    if (s.isEmpty) continue;
    var p = s.split(RegExp(r'[;\t ]+'));
    if (p.length < 3 && s.contains(',')) p = s.split(',');
    if (p.length < 3) continue;
    final x = double.tryParse(p[1].replaceAll(',', '.'));
    final y = double.tryParse(p[2].replaceAll(',', '.'));
    if (x != null && y != null) r.add(Borne(p[0], x, y));
  }
  return r;
}

double _sinh(double x) => (exp(x) - exp(-x)) / 2;
double _cosh(double x) => (exp(x) + exp(-x)) / 2;

/// UTM (WGS84 / ITRF96, écart négligeable) -> [latitude, longitude] en degrés.
List<double> utmVersLatLon(double e, double n, {int zone = 30}) {
  const a = 6378137.0, f = 1 / 298.257223563, k0 = 0.9996;
  final q = f / (2 - f), q2 = q * q, q3 = q2 * q, q4 = q3 * q;
  final aa = a / (1 + q) * (1 + q2 / 4 + q4 / 64);
  final b = [
    q / 2 - 2 * q2 / 3 + 37 * q3 / 96 - q4 / 360,
    q2 / 48 + q3 / 15 - 437 * q4 / 1440,
    17 * q3 / 480 - 37 * q4 / 840,
    4397 * q4 / 161280
  ];
  final d = [2 * q - 2 * q2 / 3 - 2 * q3, 7 * q2 / 3 - 8 * q3 / 5, 56 * q3 / 15];
  final xi = n / (k0 * aa), eta = (e - 500000) / (k0 * aa);
  var x1 = xi, e1 = eta;
  for (var j = 1; j <= 4; j++) {
    x1 -= b[j - 1] * sin(2 * j * xi) * _cosh(2 * j * eta);
    e1 -= b[j - 1] * cos(2 * j * xi) * _sinh(2 * j * eta);
  }
  final chi = asin(sin(x1) / _cosh(e1));
  final lat = chi + d[0] * sin(2 * chi) + d[1] * sin(4 * chi) + d[2] * sin(6 * chi);
  final lon = atan(_sinh(e1) / cos(x1)) + (zone * 6 - 183) * pi / 180;
  return [lat * 180 / pi, lon * 180 / pi];
}


/// Échelle approximative de la carte de situation une fois placée dans un cadre de [boxPt] points de large.
String echelleSituation(double lat, int z, {double boxPt = 150}) {
  final mParPx = 156543.03 * cos(lat * pi / 180) / pow(2, z);
  final e = mParPx * 768 / (boxPt / 2.83465 / 1000);
  return '1:${((e / 50000).round() * 50000)}';
}

/// Assemble 3x3 tuiles OpenStreetMap autour du point, ajoute un cercle rouge.
/// Hors connexion : renvoie la dernière carte mise en cache pour ce projet (ou null).
Future<Uint8List?> carteSituation(double lat, double lon, String id, {int z = 11}) async {
  final dir = await getApplicationDocumentsDirectory();
  final cache = File('${dir.path}/situation_$id.png');
  try {
    final nt = pow(2, z).toDouble();
    final px = (lon + 180) / 360 * nt;
    final r = lat * pi / 180;
    final py = (1 - log(tan(r) + 1 / cos(r)) / pi) / 2 * nt;
    final tx = px.floor(), ty = py.floor();
    final out = img.Image(width: 768, height: 768);
    for (var dy = -1; dy <= 1; dy++) {
      for (var dx = -1; dx <= 1; dx++) {
        final res = await http.get(
          Uri.parse('https://tile.openstreetmap.org/$z/${tx + dx}/${ty + dy}.png'),
          headers: {'User-Agent': 'ExtraitTopo/0.1 (usage personnel)'},
        );
        if (res.statusCode != 200) throw Exception('tuile ${res.statusCode}');
        final t = img.decodePng(res.bodyBytes);
        if (t != null) img.compositeImage(out, t, dstX: (dx + 1) * 256, dstY: (dy + 1) * 256);
      }
    }
    final cx = ((px - tx) * 256 + 256).round(), cy = ((py - ty) * 256 + 256).round();
    for (var rad = 16; rad <= 19; rad++) {
      img.drawCircle(out, x: cx, y: cy, radius: rad, color: img.ColorRgb8(220, 0, 0));
    }
    final png = Uint8List.fromList(img.encodePng(out));
    await cache.writeAsBytes(png);
    return png;
  } catch (_) {
    return await cache.exists() ? await cache.readAsBytes() : null;
  }
}


/// Génère l'extrait A3 paysage. [f] : centre, ilot, lot, iduf, tf, section, plan, livre, demandeur, lieu, date, leve.
Future<Uint8List> genererPdf(List<Borne> p, Map<String, String> f,
    {required bool dtc, Uint8List? situation, String echSit = '1:500000'}) async {
  final cal = calculer(p);
  final n = p.length, dec = dtc ? 3 : 2;
  final doc = PdfDocument();
  final reg = PdfFont.times(doc);
  final bold = PdfFont.timesBold(doc);
  final fmt = PdfPageFormat.a3.landscape;
  final g = PdfPage(doc, pageFormat: fmt).getGraphics();
  final W = fmt.width, H = fmt.height;
  g.setStrokeColor(PdfColors.black);
  g.setLineWidth(1);

  void tx(double x, double y, String s, {PdfFont? font, double z = 9, String al = 'l'}) {
    final fo = font ?? reg;
    final w = (fo.stringMetrics(s) * z).width;
    g.setFillColor(PdfColors.black);
    g.drawString(fo, z, s, al == 'c' ? x - w / 2 : al == 'r' ? x - w : x, y);
  }
  void line(double a, double b, double c, double d) {
    g.moveTo(a, b);
    g.lineTo(c, d);
    g.strokePath();
  }
  void rect(double x, double y, double w, double h) {
    g.drawRect(x, y, w, h);
    g.strokePath();
  }

  final L = 20.0, B = 20.0, T = H - 20, R = W - 20, xs = L + 470, hh = 150.0;
  rect(L, B, R - L, T - B);
  line(xs, B, xs, T);
  line(L, T - hh, xs, T - hh);
  double ty;
  List<String> foot;
  if (dtc) {
    line(xs, T - 34, R, T - 34);
    line(xs, T - hh, R, T - hh);
    tx((xs + R) / 2, T - 24, 'EXTRAIT TOPOGRAPHIQUE DE PARCELLE', z: 16, al: 'c');
    const mini = ['République de Côte d\'Ivoire', 'Ministère de l\'Urbanisme,', 'du Logement et du Cadre de Vie', '----------------', 'Direction Générale de', 'l\'Urbanisme et du Foncier', '----------------', 'Direction de la Topographie', 'et de la Cartographie'];
    for (var i = 0; i < mini.length; i++) tx(L + 235, T - 22 - i * 14, mini[i], al: 'c');
    final cols = [
      [xs + 10, ['Situation : ', 'Demandeur : ${f['demandeur']}', 'Contenance levée : ${cal.contenance}', 'N° IDUF-CI : ${f['iduf']}']],
      [xs + 250, ['Système de référence terrestre: ITRF 1996', 'Système de Référence Altimétrique: NGCI', 'Système de Représentation plane: UTM 30 N', 'Système de Coordonnées: Rectangulaires (unité mètre)']],
      [xs + 560, ['Ilot : ${f['ilot']}   Lot : ${f['lot']}', 'Immatriculation :', 'TF N° : ${f['tf']}', 'Acte N° :']],
    ];
    for (final c in cols) {
      final ls = c[1] as List<String>;
      for (var i = 0; i < ls.length; i++) tx(c[0] as double, T - 54 - i * 16, ls[i], z: 8.5);
    }
    tx(xs + 10, T - 130, '(Décret n° 2019-220 du 13/03/2019)', z: 6.5);
    ty = T - hh - 20;
    tx((xs + R) / 2, ty, 'TABLEAU DE COORDONNEES', font: bold, z: 8, al: 'c');
    ty -= 14;
    foot = ['${f['lieu']}, le ${f['date']}', 'Géomètre Assermenté de la DTC', 'Tous droit réservé N°.................../DTC/2021'];
  } else {
    final mini = ['République de Côte d\'Ivoire', '', 'Ministère des Finances et du Budget', '', 'Direction du Cadastre', '', 'Bureau de ${f['lieu']}'];
    for (var i = 0; i < mini.length; i++) tx(L + 75, T - 30 - i * 13, mini[i], al: 'c');
    line(L + 150, T - hh, L + 150, T);
    final ls = ['Centre : ${f['centre']}', 'Îlot : ${f['ilot']}    Lot : ${f['lot']}', 'N° IDUFCI : ${f['iduf']}', 'T.F. N° : ${f['tf']}    Section : ${f['section']}', 'N° du plan : ${f['plan']}', 'Livre foncier de : ${f['livre']}', 'Demandeur : ${f['demandeur']}', 'Contenance : ${cal.contenance}'];
    for (var i = 0; i < ls.length; i++) tx(L + 158, T - 18 - i * 15, ls[i], font: i >= 6 ? bold : reg, z: 8.5);
    tx((xs + R) / 2, T - 20, 'TABLEAU DE COORDONNÉES', font: bold, z: 8, al: 'c');
    tx((xs + R) / 2, T - 31, 'RECTANGULAIRES MÉTRIQUES (X, Y) DES SOMMETS', z: 7, al: 'c');
    tx((xs + R) / 2, T - 41, 'WGS 84 / UTM zone 30N', z: 7, al: 'c');
    ty = T - 56;
    foot = ['Copie certifiée conforme', '${f['lieu']}, le : ${f['date']}', 'Le Géomètre Assermenté du Cadastre'];
  }
  // tableau en deux blocs
  const cw = [38.0, 80.0, 80.0, 60.0, 62.0];
  const rh = 26.0;
  final rows = (n + 1) ~/ 2;
  for (var blk = 0; blk < 2; blk++) {
    final x = xs + 12 + blk * (cw.reduce((a, b) => a + b) + 12);
    final heads = ['Bornes', 'X (m)', 'Y (m)', 'Angles (gr)', 'Distance (m)'];
    var xx = x;
    for (var j = 0; j < 5; j++) {
      rect(xx, ty - rh, cw[j], rh);
      tx(xx + cw[j] / 2, ty - rh / 2 - 3, heads[j], font: bold, z: 8, al: 'c');
      xx += cw[j];
    }
    for (var r = 0; r < rows; r++) {
      final i = blk * rows + r;
      if (i >= n) break;
      final b = p[i];
      final vals = [b.n, b.x.toStringAsFixed(dec), b.y.toStringAsFixed(dec), b.g.toStringAsFixed(dec), b.d.toStringAsFixed(dec)];
      xx = x;
      for (var j = 0; j < 5; j++) {
        rect(xx, ty - rh * (r + 2), cw[j], rh);
        tx(j == 0 ? xx + cw[j] / 2 : xx + cw[j] - 4, ty - rh * (r + 2) + rh / 2 - 3, vals[j], z: 8.5, al: j == 0 ? 'c' : 'r');
        xx += cw[j];
      }
    }
  }
  rect(R - 90, B + 12, 76, 76);
  tx(R - 52, B + 46, 'QR code', z: 8, al: 'c');
  // plan
  final x0 = L, y0 = B, x1 = xs, y1 = T - hh;
  if (situation != null) {
    g.drawImage(PdfImage.file(doc, bytes: situation), x0 + 10, y1 - 105, 150, 95);
  }
  rect(x0 + 10, y1 - 105, 150, 95);
  tx(x0 + 85, y1 - 118, echSit, z: 8, al: 'c');
  final ax = x0 + (x1 - x0) * 0.66;
  line(ax, y1 - 70, ax, y1 - 25);
  line(ax, y1 - 25, ax - 5, y1 - 37);
  line(ax, y1 - 25, ax + 5, y1 - 37);
  tx(ax, y1 - 82, 'N', z: 10, al: 'c');
  final xmin = x0 + 45, xmax = x1 - 45, ymin = y0 + 85, ymax = y1 - 135;
  final xsx = p.map((e) => e.x), ysy = p.map((e) => e.y);
  final ex = xsx.reduce(max) - xsx.reduce(min), ey = ysy.reduce(max) - ysy.reduce(min);
  var N = 500;
  for (final c in [500, 1000, 2000, 2500, 5000, 10000, 20000, 25000, 50000, 100000]) {
    N = c;
    if (ex * 2.83465 * 1000 / N <= xmax - xmin && ey * 2.83465 * 1000 / N <= ymax - ymin) break;
  }
  final k = 2.83465 * 1000 / N;
  final xc = (xsx.reduce(max) + xsx.reduce(min)) / 2, yc = (ysy.reduce(max) + ysy.reduce(min)) / 2;
  final cx = (xmin + xmax) / 2, cy = (ymin + ymax) / 2;
  double X(double x) => cx + (x - xc) * k;
  double Y(double y) => cy + (y - yc) * k;
  final st = N / 10;
  final gx0 = (((x0 + 35 - cx) / k + xc) / st).ceil() * st;
  for (var gx = gx0; X(gx) < x1 - 35; gx += st) {
    tx(X(gx), y1 - 12, gx.round().toString(), z: 6.5, al: 'c');
    line(X(gx), y1 - 16, X(gx), y1 - 22);
  }
  for (var gy = (((y0 + 40 - cy) / k + yc) / st).ceil() * st; Y(gy) < y1 - 140; gy += st) {
    tx(x0 + 8, Y(gy), gy.round().toString(), z: 6.5);
    line(x0 + 36, Y(gy), x0 + 42, Y(gy));
    for (var gx = gx0; X(gx) < x1 - 35; gx += st) {
      line(X(gx) - 3, Y(gy), X(gx) + 3, Y(gy));
      line(X(gx), Y(gy) - 3, X(gx), Y(gy) + 3);
    }
  }
  g.setLineWidth(1.1);
  g.moveTo(X(p[0].x), Y(p[0].y));
  for (var i = 1; i < n; i++) g.lineTo(X(p[i].x), Y(p[i].y));
  g.closePath();
  g.strokePath();
  for (final b in p) {
    final px = X(b.x), py = Y(b.y), dx = px - cx, dy = py - cy;
    final m = max(sqrt(dx * dx + dy * dy), 1e-6);
    g.setFillColor(PdfColors.white);
    g.drawEllipse(px, py, 2.2, 2.2);
    g.fillAndStrokePath();
    tx(px + dx / m * 10, py + dy / m * 10 - 2.5, b.n, z: 6.5, al: 'c');
  }
  for (var i = 0; i < foot.length; i++) tx(x0 + 14, y0 + 50 - i * 11, foot[i], z: 8);
  tx(x1 - 14, y0 + 50, 'Levé et dressé par ${f['leve']}', z: 7, al: 'r');
  tx((x0 + x1) / 2, y0 + 10, '1:$N', z: 9, al: 'c');
  return await doc.save();
}


void main() => runApp(const MaterialApp(home: Accueil()));

class Accueil extends StatefulWidget {
  const Accueil({super.key});
  @override
  State<Accueil> createState() => _AccueilState();
}

class _AccueilState extends State<Accueil> {
  static const noms = {
    'centre': 'Centre', 'ilot': 'Îlot', 'lot': 'Lot', 'iduf': 'N° IDUF-CI', 'tf': 'T.F. N°',
    'section': 'Section', 'plan': 'N° du plan', 'livre': 'Livre foncier de', 'demandeur': 'Demandeur',
    'lieu': 'Lieu', 'date': 'Date', 'leve': 'Levé et dressé par',
  };
  final champs = {for (final k in noms.keys) k: TextEditingController()};
  final pts = TextEditingController();
  bool dtc = false;
  Uint8List? sit;
  String ech = '1:500000', info = 'Importez ou collez vos bornes (nom;X;Y).';

  void maj() {
    final b = lireBornes(pts.text);
    if (b.length < 3) return setState(() => info = 'Minimum 3 bornes.');
    final c = calculer(b);
    setState(() => info = '${b.length} bornes · ${c.contenance} · périmètre ${c.perimetre.toStringAsFixed(2)} m');
  }

  Future<void> importer() async {
    final r = await FilePicker.platform.pickFiles(withData: true);
    if (r?.files.single.bytes == null) return;
    pts.text = String.fromCharCodes(r!.files.single.bytes!);
    maj();
  }

  Future<void> situationAuto() async {
    final b = lireBornes(pts.text);
    if (b.length < 3) return;
    final ll = utmVersLatLon(b.map((e) => e.x).reduce((a, c) => a + c) / b.length,
        b.map((e) => e.y).reduce((a, c) => a + c) / b.length);
    final png = await carteSituation(ll[0], ll[1], champs['plan']!.text.isEmpty ? 'projet' : champs['plan']!.text);
    setState(() {
      sit = png;
      ech = echelleSituation(ll[0], 11);
      info = png == null ? 'Pas de réseau ni de carte en cache : choisissez une image.' : 'Carte de situation prête.';
    });
  }

  Future<void> situationImage() async {
    final r = await FilePicker.platform.pickFiles(type: FileType.image, withData: true);
    if (r?.files.single.bytes != null) setState(() => sit = r!.files.single.bytes);
  }

  Future<void> exporter() async {
    final b = lireBornes(pts.text);
    if (b.length < 3) return;
    final f = {for (final e in champs.entries) e.key: e.value.text};
    final pdf = await genererPdf(b, f, dtc: dtc, situation: sit, echSit: ech);
    await Printing.sharePdf(bytes: pdf, filename: dtc ? 'extrait_dtc.pdf' : 'extrait_cadastre.pdf');
  }

  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: const Text('Extrait topographique')),
        body: ListView(padding: const EdgeInsets.all(12), children: [
          SegmentedButton<bool>(
            segments: const [ButtonSegment(value: false, label: Text('Cadastre')), ButtonSegment(value: true, label: Text('DTC'))],
            selected: {dtc},
            onSelectionChanged: (s) => setState(() => dtc = s.first),
          ),
          for (final e in noms.entries)
            TextField(controller: champs[e.key], decoration: InputDecoration(labelText: e.value)),
          const SizedBox(height: 12),
          TextField(controller: pts, maxLines: 8, onChanged: (_) => maj(), decoration: const InputDecoration(labelText: 'Bornes : nom;X;Y', border: OutlineInputBorder())),
          Text(info),
          Wrap(spacing: 8, children: [
            OutlinedButton(onPressed: importer, child: const Text('Importer CSV')),
            OutlinedButton(onPressed: situationAuto, child: const Text('Carte de situation')),
            OutlinedButton(onPressed: situationImage, child: const Text('Choisir une image')),
          ]),
          const SizedBox(height: 12),
          FilledButton(onPressed: exporter, child: const Text('Générer le PDF (A3)')),
        ]),
      );
}
