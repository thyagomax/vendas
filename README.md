<?php
session_start();

// Habilitar exibição de erros para depuração (mas será controlada no upload)
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);

// Verificar se o usuário está logado
if (!isset($_SESSION['loggedin']) || $_SESSION['loggedin'] !== true) {
    header("Location: login.php");
    exit;
}

// Processar logout
if (isset($_GET['logout'])) {
    session_destroy();
    header("Location: login.php");
    exit;
}

// Função para obter as iniciais do nome
function getInitials($name) {
    $words = explode(' ', trim($name));
    $initials = '';
    foreach ($words as $word) {
        if (!empty($word)) {
            $initials .= strtoupper(substr($word, 0, 1));
        }
        if (strlen($initials) >= 2) break;
    }
    return $initials ?: strtoupper(substr($name, 0, 1));
}

$initials = getInitials($_SESSION['full_name']);

// Função para obter o primeiro e segundo nome
function getFirstAndSecondName($full_name) {
    $words = explode(' ', trim($full_name));
    $first_name = $words[0] ?? '';
    $second_name = $words[1] ?? '';
    return "$first_name $second_name";
}

$display_name = getFirstAndSecondName($_SESSION['full_name']);

// Conexão com o banco de dados e criação das tabelas
$dbPath = __DIR__ . '/database.db';
try {
    // Verificar permissões do diretório
    $dbDir = dirname($dbPath);
    if (!is_writable($dbDir)) {
        $_SESSION['error'] = "O diretório '$dbDir' não tem permissões de escrita. Por favor, ajuste as permissões do diretório.";
        header("Location: ?page=dashboard");
        exit;
    }

    $db = new PDO("sqlite:$dbPath");
    $db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

    // Criar tabelas se não existirem
    $db->exec("CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        full_name TEXT NOT NULL,
        username TEXT NOT NULL UNIQUE,
        email TEXT NOT NULL,
        phone TEXT,
        company_name TEXT,
        password TEXT NOT NULL
    )");

    $db->exec("CREATE TABLE IF NOT EXISTS user_settings (
        user_id INTEGER PRIMARY KEY,
        system_name TEXT,
        profile_photo TEXT,
        FOREIGN KEY (user_id) REFERENCES users(id)
    )");

    $db->exec("CREATE TABLE IF NOT EXISTS clientes (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER NOT NULL,
        nome TEXT NOT NULL,
        telefone TEXT,
        endereco TEXT,
        numero TEXT,
        bairro TEXT,
        cidade TEXT,
        FOREIGN KEY (user_id) REFERENCES users(id)
    )");

    $db->exec("CREATE TABLE IF NOT EXISTS produtos (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER NOT NULL,
        nome TEXT NOT NULL,
        preco REAL NOT NULL,
        estoque INTEGER NOT NULL,
        foto TEXT,
        FOREIGN KEY (user_id) REFERENCES users(id)
    )");

    $db->exec("CREATE TABLE IF NOT EXISTS vendas (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER NOT NULL,
        cliente_id INTEGER,
        produto_id INTEGER,
        quantidade INTEGER NOT NULL,
        data_entrega DATE,
        valor_total REAL,
        FOREIGN KEY (user_id) REFERENCES users(id),
        FOREIGN KEY (cliente_id) REFERENCES clientes(id),
        FOREIGN KEY (produto_id) REFERENCES produtos(id)
    )");

    $db->exec("CREATE TABLE IF NOT EXISTS pagamentos (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER NOT NULL,
        venda_id INTEGER,
        valor REAL,
        data_pagamento DATE,
        FOREIGN KEY (user_id) REFERENCES users(id),
        FOREIGN KEY (venda_id) REFERENCES vendas(id)
    )");

    // Verificar se a coluna 'quantidade' existe na tabela 'vendas'
    $stmt = $db->query("PRAGMA table_info(vendas)");
    $columns = $stmt->fetchAll(PDO::FETCH_ASSOC);
    $hasQuantidade = false;
    foreach ($columns as $column) {
        if ($column['name'] === 'quantidade') {
            $hasQuantidade = true;
            break;
        }
    }

    // Se a coluna 'quantidade' não existe, adicioná-la
    if (!$hasQuantidade) {
        $db->exec("ALTER TABLE vendas ADD COLUMN quantidade INTEGER NOT NULL DEFAULT 1");
    }
	
	// Verificar se a coluna 'user_id' existe nas tabelas
$tables = ['clientes', 'produtos', 'vendas', 'pagamentos'];
foreach ($tables as $table) {
    $stmt = $db->query("PRAGMA table_info($table)");
    $columns = $stmt->fetchAll(PDO::FETCH_ASSOC);
    $hasUserId = false;
    foreach ($columns as $column) {
        if ($column['name'] === 'user_id') {
            $hasUserId = true;
            break;
        }
    }
    if (!$hasUserId) {
        $db->exec("ALTER TABLE $table ADD COLUMN user_id INTEGER NOT NULL DEFAULT 0");
    }
}
// Verificar e adicionar coluna 'preco_venda' na tabela 'produtos'
$stmt = $db->query("PRAGMA table_info(produtos)");
$columns = $stmt->fetchAll(PDO::FETCH_ASSOC);
$hasPrecoVenda = false;
foreach ($columns as $column) {
    if ($column['name'] === 'preco_venda') {
        $hasPrecoVenda = true;
        break;
    }
}
if (!$hasPrecoVenda) {
    $db->exec("ALTER TABLE produtos ADD COLUMN preco_venda REAL");
    // Preencher preco_venda com o valor de preco para registros existentes
    $db->exec("UPDATE produtos SET preco_venda = preco WHERE preco_venda IS NULL");
}

// Verificar e adicionar coluna 'forma_pagamento' na tabela 'vendas'
$stmt = $db->query("PRAGMA table_info(vendas)");
$columns = $stmt->fetchAll(PDO::FETCH_ASSOC);
$hasFormaPagamento = false;
foreach ($columns as $column) {
    if ($column['name'] === 'forma_pagamento') {
        $hasFormaPagamento = true;
        break;
    }
}
if (!$hasFormaPagamento) {
    $db->exec("ALTER TABLE vendas ADD COLUMN forma_pagamento TEXT");
}

// Verificar e adicionar coluna 'desconto' na tabela 'vendas'
$stmt = $db->query("PRAGMA table_info(vendas)");
$columns = $stmt->fetchAll(PDO::FETCH_ASSOC);
$hasDesconto = false;
foreach ($columns as $column) {
    if ($column['name'] === 'desconto') {
        $hasDesconto = true;
        break;
    }
}
if (!$hasDesconto) {
    $db->exec("ALTER TABLE vendas ADD COLUMN desconto REAL DEFAULT 0");
}

} catch (Exception $e) {
    $_SESSION['error'] = "Erro na inicialização do banco de dados: " . $e->getMessage();
    header("Location: ?page=dashboard");
    exit;
}

// Obter o user_id do usuário logado
$user_id = null;
try {
    $stmt = $db->prepare("SELECT id FROM users WHERE username = :username");
    $stmt->execute([':username' => $_SESSION['username']]);
    $user = $stmt->fetch(PDO::FETCH_ASSOC);
    if (!$user) {
        throw new Exception("Usuário não encontrado.");
    }
    $user_id = $user['id'];
} catch (Exception $e) {
    $_SESSION['error'] = "Erro ao obter ID do usuário: " . $e->getMessage();
    header("Location: ?page=dashboard");
    exit;
}

// Obter o nome personalizado do sistema e a foto de perfil do usuário
$system_name = "Altere seu nome"; // Nome padrão
$profile_photo = null;
try {
    $stmt = $db->prepare("SELECT system_name, profile_photo FROM user_settings WHERE user_id = :user_id");
    $stmt->execute([':user_id' => $user_id]);
    $settings = $stmt->fetch(PDO::FETCH_ASSOC);
    if ($settings) {
        if (!empty($settings['system_name']) && $_SESSION['username'] !== 'thyago') {
            $system_name = $settings['system_name'];
        }
        $profile_photo = $settings['profile_photo'];
    }
} catch (PDOException $e) {
    $_SESSION['error'] = "Erro ao carregar configurações do usuário: " . $e->getMessage();
    header("Location: ?page=dashboard");
    exit;
}

// Processar formulários
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    try {
        // Excluir Usuário (Admin)
        if (isset($_POST['delete_user']) && $_SESSION['username'] === 'thyago') {
            $user_id_to_delete = $_POST['id'];
            $stmt = $db->prepare("DELETE FROM users WHERE id = :id");
            $stmt->execute([':id' => $user_id_to_delete]);
            $_SESSION['success'] = "Usuário excluído com sucesso!";
            header("Location: ?page=admin");
            exit;
        }

        // Atualizar Nome do Sistema
        if (isset($_POST['update_system_name']) && $_SESSION['username'] !== 'thyago') {
            $new_system_name = trim($_POST['system_name']);
            if (!empty($new_system_name)) {
                $stmt = $db->prepare("INSERT OR REPLACE INTO user_settings (user_id, system_name, profile_photo) VALUES (:user_id, :system_name, :profile_photo)");
                $stmt->execute([
                    ':user_id' => $user_id,
                    ':system_name' => $new_system_name,
                    ':profile_photo' => $settings['profile_photo'] ?? null
                ]);
                $_SESSION['success'] = "Nome do sistema atualizado com sucesso!";
                $system_name = $new_system_name;
            } else {
                $_SESSION['error'] = "O nome do sistema não pode estar vazio.";
            }
            header("Location: ?page=settings");
            exit;
        }

        // Remover a Foto de Perfil
        if (isset($_POST['remove_profile_photo']) && $_SESSION['username'] !== 'thyago') {
            // Excluir a foto do servidor, se existir
            if ($profile_photo && file_exists($profile_photo)) {
                unlink($profile_photo);
            }

            // Atualizar o banco de dados para remover a foto
            $stmt = $db->prepare("UPDATE user_settings SET profile_photo = NULL WHERE user_id = :user_id");
            $stmt->execute([':user_id' => $user_id]);
            $profile_photo = null;
            $_SESSION['success'] = "Foto de perfil removida com sucesso!";
            header("Location: ?page=settings");
            exit;
        }

        // Upload da Foto de Perfil (na página de configurações)
        if (isset($_POST['upload_profile_photo']) && $_SESSION['username'] !== 'thyago') {
            // Suprimir avisos do PHP durante o upload para evitar saídas prematuras
            $original_error_level = error_reporting();
            error_reporting(0);

            try {
                // Verificar erros de upload
                switch ($_FILES['profile_photo']['error']) {
                    case UPLOAD_ERR_INI_SIZE:
                    case UPLOAD_ERR_FORM_SIZE:
                        throw new Exception("O arquivo excede o tamanho máximo permitido pelo servidor.");
                    case UPLOAD_ERR_NO_FILE:
                        throw new Exception("Nenhum arquivo foi enviado.");
                    case UPLOAD_ERR_OK:
                        break;
                    default:
                        throw new Exception("Erro ao carregar a foto de perfil.");
                }

                // Definir limite de tamanho (2MB)
                $maxFileSize = 2 * 1024 * 1024; // 2MB em bytes
                if ($_FILES['profile_photo']['size'] > $maxFileSize) {
                    throw new Exception("O arquivo excede o tamanho máximo permitido de 2MB.");
                }

                // Validar tipo de arquivo (apenas imagens)
                $allowed_types = ['image/jpeg', 'image/png', 'image/gif'];
                $file_type = mime_content_type($_FILES['profile_photo']['tmp_name']);
                if (!in_array($file_type, $allowed_types)) {
                    throw new Exception("Apenas arquivos JPEG, PNG ou GIF são permitidos.");
                }

                // Salvar a nova foto
                $uploadDir = 'images/';
                if (!is_dir($uploadDir)) {
                    mkdir($uploadDir, 0755, true);
                }
                $photoName = uniqid() . '-' . basename($_FILES['profile_photo']['name']);
                $photoPath = $uploadDir . $photoName;

                if (!move_uploaded_file($_FILES['profile_photo']['tmp_name'], $photoPath)) {
                    throw new Exception("Erro ao fazer upload da foto de perfil.");
                }

                // Excluir a foto antiga, se existir
                if ($profile_photo && file_exists($profile_photo)) {
                    unlink($profile_photo);
                }

                // Atualizar o caminho da foto no banco
                $stmt = $db->prepare("INSERT OR REPLACE INTO user_settings (user_id, system_name, profile_photo) VALUES (:user_id, :system_name, :profile_photo)");
                $stmt->execute([
                    ':user_id' => $user_id,
                    ':system_name' => $settings['system_name'] ?? null,
                    ':profile_photo' => $photoPath
                ]);

                // Atualizar a variável para exibir a nova foto
                $profile_photo = $photoPath;
                $_SESSION['success'] = "Foto de perfil atualizada com sucesso!";
            } catch (Exception $e) {
                $_SESSION['error'] = "Erro ao processar formulário: " . $e->getMessage();
            } finally {
                // Restaurar o nível de erro original
                error_reporting($original_error_level);
            }

            header("Location: ?page=settings");
            exit;
        }

        // Adicionar Cliente
        if (isset($_POST['add_cliente'])) {
            $stmt = $db->prepare("INSERT INTO clientes (user_id, nome, telefone, endereco, numero, bairro, cidade) VALUES (:user_id, :nome, :telefone, :endereco, :numero, :bairro, :cidade)");
            $stmt->execute([
                ':user_id' => $user_id,
                ':nome' => $_POST['nome'],
                ':telefone' => $_POST['telefone'],
                ':endereco' => $_POST['endereco'],
                ':numero' => $_POST['numero'],
                ':bairro' => $_POST['bairro'],
                ':cidade' => $_POST['cidade']
            ]);
            $_SESSION['success'] = "Cliente cadastrado com sucesso!";
            header("Location: ?page=clientes");
            exit;
        }

        // Editar Cliente
        if (isset($_POST['edit_cliente'])) {
            $stmt = $db->prepare("UPDATE clientes SET nome = :nome, telefone = :telefone, endereco = :endereco, numero = :numero, bairro = :bairro, cidade = :cidade WHERE id = :id AND user_id = :user_id");
            $stmt->execute([
                ':id' => $_POST['id'],
                ':user_id' => $user_id,
                ':nome' => $_POST['nome'],
                ':telefone' => $_POST['telefone'],
                ':endereco' => $_POST['endereco'],
                ':numero' => $_POST['numero'],
                ':bairro' => $_POST['bairro'],
                ':cidade' => $_POST['cidade']
            ]);
            $_SESSION['success'] = "Cliente atualizado com sucesso!";
            header("Location: ?page=clientes");
            exit;
        }

        // Excluir Cliente
        if (isset($_POST['delete_cliente'])) {
            $stmt = $db->prepare("DELETE FROM clientes WHERE id = :id AND user_id = :user_id");
            $stmt->execute([':id' => $_POST['id'], ':user_id' => $user_id]);
            $_SESSION['success'] = "Cliente excluído com sucesso!";
            header("Location: ?page=clientes");
            exit;
        }

        // Adicionar Produto
        if (isset($_POST['add_produto'])) {
    $fotoPath = null;
    if (isset($_FILES['foto']) && $_FILES['foto']['error'] === UPLOAD_ERR_OK) {
        $uploadDir = 'images/';
        if (!is_dir($uploadDir)) {
            mkdir($uploadDir, 0755, true);
        }
        $fotoName = uniqid() . '-' . basename($_FILES['foto']['name']);
        $fotoPath = $uploadDir . $fotoName;
        if (!move_uploaded_file($_FILES['foto']['tmp_name'], $fotoPath)) {
            $_SESSION['error'] = "Erro ao fazer upload da imagem.";
            $fotoPath = null;
        }
    }

    $stmt = $db->prepare("INSERT INTO produtos (user_id, nome, preco, preco_venda, estoque, foto) VALUES (:user_id, :nome, :preco, :preco_venda, :estoque, :foto)");
    $stmt->execute([
        ':user_id' => $user_id,
        ':nome' => $_POST['nome'],
        ':preco' => $_POST['preco'],
        ':preco_venda' => $_POST['preco_venda'],
        ':estoque' => $_POST['estoque'],
        ':foto' => $fotoPath
    ]);

    if ($stmt->rowCount() > 0) {
        $_SESSION['success'] = "Produto cadastrado com sucesso!";
    } else {
        $_SESSION['error'] = "Erro ao cadastrar produto.";
    }
    header("Location: ?page=produtos");
    exit;
}

        // Editar Produto
        if (isset($_POST['edit_produto'])) {
    $fotoPath = $_POST['foto_atual'];
    if (isset($_FILES['foto']) && $_FILES['foto']['error'] === UPLOAD_ERR_OK) {
        // Excluir a foto antiga, se existir
        if ($fotoPath && file_exists($fotoPath)) {
            unlink($fotoPath);
        }
        $uploadDir = 'images/';
        if (!is_dir($uploadDir)) {
            mkdir($uploadDir, 0755, true);
        }
        $fotoName = uniqid() . '-' . basename($_FILES['foto']['name']);
        $fotoPath = $uploadDir . $fotoName;
        if (!move_uploaded_file($_FILES['foto']['tmp_name'], $fotoPath)) {
            $_SESSION['error'] = "Erro ao fazer upload da nova imagem.";
            $fotoPath = $_POST['foto_atual'];
        }
    }

    $stmt = $db->prepare("UPDATE produtos SET nome = :nome, preco = :preco, preco_venda = :preco_venda, estoque = :estoque, foto = :foto WHERE id = :id AND user_id = :user_id");
    $stmt->execute([
        ':id' => $_POST['id'],
        ':user_id' => $user_id,
        ':nome' => $_POST['nome'],
        ':preco' => $_POST['preco'],
        ':preco_venda' => $_POST['preco_venda'],
        ':estoque' => $_POST['estoque'],
        ':foto' => $fotoPath
    ]);

    if ($stmt->rowCount() > 0) {
        $_SESSION['success'] = "Produto atualizado com sucesso!";
    } else {
        $_SESSION['error'] = "Erro ao atualizar produto ou produto não encontrado.";
    }
    header("Location: ?page=produtos");
    exit;
}

        // Excluir Produto
        if (isset($_POST['delete_produto'])) {
            // Buscar o caminho da foto para excluí-la
            $stmt = $db->prepare("SELECT foto FROM produtos WHERE id = :id AND user_id = :user_id");
            $stmt->execute([':id' => $_POST['id'], ':user_id' => $user_id]);
            $produto = $stmt->fetch(PDO::FETCH_ASSOC);
            if ($produto['foto'] && file_exists($produto['foto'])) {
                unlink($produto['foto']);
            }

            $stmt = $db->prepare("DELETE FROM produtos WHERE id = :id AND user_id = :user_id");
            $stmt->execute([':id' => $_POST['id'], ':user_id' => $user_id]);
            $_SESSION['success'] = "Produto excluído com sucesso!";
            header("Location: ?page=produtos");
            exit;
        }

        // Adicionar Venda
        if (isset($_POST['add_venda'])) {
            // Iniciar uma transação para garantir consistência
            $db->beginTransaction();

            try {
                $quantidade = (int)$_POST['quantidade'];
                if ($quantidade <= 0) {
                    throw new Exception("A quantidade deve ser maior que zero.");
                }

                // Inserir a venda
                $stmt = $db->prepare("INSERT INTO vendas (user_id, cliente_id, produto_id, quantidade, forma_pagamento, desconto, data_entrega, valor_total) VALUES (:user_id, :cliente_id, :produto_id, :quantidade, :forma_pagamento, :desconto, :data_entrega, :valor_total)");
$stmt->execute([
    ':user_id' => $user_id,
    ':cliente_id' => $_POST['cliente_id'],
    ':produto_id' => $_POST['produto_id'],
    ':quantidade' => $quantidade,
    ':forma_pagamento' => $_POST['forma_pagamento'],
    ':desconto' => $_POST['desconto'] ? floatval($_POST['desconto']) : 0,
    ':data_entrega' => $_POST['data_entrega'],
    ':valor_total' => $_POST['valor_total']
]);

                // Atualizar o estoque do produto
                $stmt = $db->prepare("UPDATE produtos SET estoque = estoque - :quantidade WHERE id = :produto_id AND user_id = :user_id AND estoque >= :quantidade");
                $stmt->execute([
                    ':quantidade' => $quantidade,
                    ':produto_id' => $_POST['produto_id'],
                    ':user_id' => $user_id
                ]);

                // Verificar se o estoque foi atualizado
                if ($stmt->rowCount() === 0) {
                    throw new Exception("Estoque insuficiente ou produto não encontrado.");
                }

                // Confirmar a transação
                $db->commit();
                $_SESSION['success'] = "Venda registrada com sucesso!";
            } catch (Exception $e) {
                // Desfazer a transação em caso de erro
                $db->rollBack();
                $_SESSION['error'] = "Erro ao registrar venda: " . $e->getMessage();
            }

            header("Location: ?page=vendas");
            exit;
        }

        // Adicionar Pagamento
        if (isset($_POST['add_pagamento'])) {
            $stmt = $db->prepare("INSERT INTO pagamentos (user_id, venda_id, valor, data_pagamento) VALUES (:user_id, :venda_id, :valor, :data_pagamento)");
            $stmt->execute([
                ':user_id' => $user_id,
                ':venda_id' => $_POST['venda_id'],
                ':valor' => $_POST['valor'],
                ':data_pagamento' => $_POST['data_pagamento']
            ]);
            $_SESSION['success'] = "Pagamento lançado com sucesso!";
            header("Location: ?page=vendas");
            exit;
        }
		
		// Gerar Relatório
if (isset($_POST['gerar_relatorio'])) {
    $_SESSION['report_data'] = [
        'tipo' => $_POST['report_type'],
        'dia' => $_POST['report_day'] ?? '',
        'mes' => $_POST['report_month'] ?? '',
        'ano' => $_POST['report_year'] ?? ''
    ];
    header("Location: ?page=relatorios");
    exit;
}

    } catch (Exception $e) {
        $_SESSION['error'] = "Erro ao processar formulário: " . $e->getMessage();
        error_log("Erro ao processar formulário em index.php: " . $e->getMessage());
        header("Location: ?page=" . urlencode($page));
        exit;
    }
}

// Exibir mensagens de sucesso ou erro
$success_message = isset($_SESSION['success']) ? $_SESSION['success'] : '';
$error_message = isset($_SESSION['error']) ? $_SESSION['error'] : '';
unset($_SESSION['success']);
unset($_SESSION['error']);
$page = isset($_GET['page']) ? $_GET['page'] : 'dashboard';
?>

<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mk - Sistemas</title>
    <script src="https://cdn.tailwindcss.com"></script>
	<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body class="bg-gray-100 min-h-screen flex flex-col">
    <!-- Barra Superior -->
    <div class="bg-gray-800 text-white p-4 flex items-center justify-between fixed top-0 left-0 right-0 z-20">
        <div class="text-lg font-semibold"></div> <!-- Nome removido da barra superior -->
        <div class="flex items-center gap-4">
            <!-- Menu Dropdown com Nome do Usuário -->
            <div class="relative flex items-center gap-2">
                <div class="w-8 h-8 bg-gray-600 rounded-full flex items-center justify-center text-white font-bold text-sm">
                    <?php echo htmlspecialchars($initials); ?>
                </div>
                <button id="userMenuButton" class="flex items-center gap-2 focus:outline-none">
                    <span><?php echo htmlspecialchars($display_name); ?></span>
                    <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"></path>
                    </svg>
                </button>
                <div id="userMenu" class="absolute right-0 mt-2 w-48 bg-gray-700 rounded shadow-lg hidden top-full">
                    <a href="?logout=true" class="block px-4 py-2 text-white hover:bg-gray-600">Sair</a>
                </div>
            </div>
        </div>
    </div>

    <!-- Botão de Seta para Mostrar a Barra (inicialmente oculto) -->
    <button id="showSidebar" class="hidden fixed top-16 left-4 text-white bg-gray-800 p-2 rounded-full hover:bg-gray-700 z-10">
        ➡️
    </button>

    <!-- Conteúdo Principal -->
    <div class="flex flex-1 pt-16">
        <!-- Barra Lateral -->
		<div id="sidebar" class="bg-gray-800 text-white w-64 min-h-screen p-4">
            <div class="flex flex-col items-center mb-6">
                <div class="w-[150px] h-[150px] rounded-full bg-gray-600 flex items-center justify-center mx-auto relative">
                    <?php if ($profile_photo): ?>
                        <img src="<?php echo htmlspecialchars($profile_photo); ?>" alt="Foto de Perfil" class="w-full h-full rounded-full object-cover">
                    <?php else: ?>
                        <span class="text-5xl font-bold"><?php echo htmlspecialchars($initials); ?></span>
                    <?php endif; ?>
                </div>
                <div class="mt-2 text-lg font-semibold text-center"><?php echo htmlspecialchars($system_name); ?></div>
            </div>
            <nav>
                <ul>
                    <li class="mb-2"><a href="?page=dashboard" class="flex items-center p-2 bg-blue-600 rounded"><span class="mr-2">📊</span> Dashboard</a></li>
                    <li class="mb-2"><a href="?page=clientes" class="flex items-center p-2 hover:bg-gray-700 rounded"><span class="mr-2">👥</span> Clientes</a></li>
                    <li class="mb-2"><a href="?page=produtos" class="flex items-center p-2 hover:bg-gray-700 rounded"><span class="mr-2">📦</span> Produtos</a></li>
<li class="mb-2"><a href="?page=vendas" class="flex items-center p-2 hover:bg-gray-700 rounded"><span class="mr-2">💸</span> Vendas</a></li>
<li class="mb-2"><a href="?page=relatorios" class="flex items-center p-2 <?php echo $page === 'relatorios' ? 'bg-blue-600' : 'hover:bg-gray-700'; ?> rounded"><span class="mr-2">📈</span> Relatórios</a></li>

<?php if ($_SESSION['username'] === 'thyago'): ?>
    <li class="mb-2"><a href="?page=admin" class="flex items-center p-2 hover:bg-gray-700 rounded"><span class="mr-2">🔧</span> Admin</a></li>
<?php else: ?>
    <li class="mb-2"><a href="?page=settings" class="flex items-center p-2 hover:bg-gray-700 rounded"><span class="mr-2">⚙️</span> Configurações</a></li>
<?php endif; ?>
                    <li class="mt-4">
                        <button id="hideSidebar" class="w-full text-left p-2 bg-gray-600 hover:bg-gray-700 rounded flex items-center">
                            <span class="mr-2">⬅️</span> Ocultar barra
                        </button>
                    </li>
                </ul>
            </nav>
        </div>

        <!-- Área Principal -->
		<!-- Página atual: <?php echo htmlspecialchars($page); ?> -->
        <div id="mainContent" class="flex-1 p-6">
            <?php if ($success_message): ?>
                <div class="bg-green-500 text-white p-3 rounded mb-4 text-center"><?php echo htmlspecialchars($success_message); ?></div>
            <?php endif; ?>
            <?php if ($error_message): ?>
                <div class="bg-red-500 text-white p-3 rounded mb-4 text-center"><?php echo htmlspecialchars($error_message); ?></div>
            <?php endif; ?>

            <?php
                        
            switch ($page) {
                case 'dashboard':
                    // Cálculo das métricas para os cards
                    try {
                        $stmt = $db->prepare("SELECT COUNT(*) FROM clientes WHERE user_id = :user_id");
                        $stmt->execute([':user_id' => $user_id]);
                        $total_clientes = $stmt->fetchColumn();

                        $stmt = $db->prepare("SELECT COUNT(*) FROM vendas WHERE user_id = :user_id");
                        $stmt->execute([':user_id' => $user_id]);
                        $total_vendas = $stmt->fetchColumn();

                        $stmt = $db->prepare("SELECT SUM(pendente) FROM (SELECT v.id, (v.valor_total - COALESCE(SUM(p.valor), 0)) AS pendente FROM vendas v LEFT JOIN pagamentos p ON v.id = p.venda_id WHERE v.user_id = :user_id GROUP BY v.id HAVING v.valor_total > COALESCE(SUM(p.valor), 0)) AS subquery");
                        $stmt->execute([':user_id' => $user_id]);
                        $total_pagamentos_pendentes = $stmt->fetchColumn() ?? 0;

                        $stmt = $db->prepare("SELECT SUM(estoque) FROM produtos WHERE user_id = :user_id");
                        $stmt->execute([':user_id' => $user_id]);
                        $total_estoque = $stmt->fetchColumn();
                    } catch (PDOException $e) {
                        $_SESSION['error'] = "Erro ao calcular métricas: " . $e->getMessage();
                        error_log("Erro ao calcular métricas em index.php: " . $e->getMessage());
                        header("Location: ?page=dashboard");
                        exit;
                    }
                    ?>
                    <!-- Cards Superiores -->
                    <div class="grid grid-cols-1 md:grid-cols-4 gap-4 mb-6">
                        <div class="bg-teal-500 text-white p-4 rounded shadow">
                            <h3 class="text-lg">Total Clientes</h3>
                            <p class="text-3xl"><?php echo $total_clientes; ?></p>
                            <a href="?page=clientes" class="text-sm underline">Mais Informações</a>
                        </div>
                        <div class="bg-green-500 text-white p-4 rounded shadow">
                            <h3 class="text-lg">Total Vendas</h3>
                            <p class="text-3xl"><?php echo $total_vendas; ?></p>
                            <a href="?page=vendas" class="text-sm underline">Mais Informações</a>
                        </div>
                        <div class="bg-yellow-500 text-white p-4 rounded shadow">
                            <h3 class="text-lg">Pagamentos Pendentes</h3>
                            <p class="text-3xl">R$ <?php echo number_format($total_pagamentos_pendentes, 2, ',', '.'); ?></p>
                            <a href="?page=vendas" class="text-sm underline">Mais Informações</a>
                        </div>
                        <div class="bg-red-500 text-white p-4 rounded shadow">
                            <h3 class="text-lg">Total Estoque</h3>
                            <p class="text-3xl"><?php echo $total_estoque; ?></p>
                            <a href="?page=produtos" class="text-sm underline">Mais Informações</a>
                        </div>
                    </div>
					
					

                    <!-- Seção Principal (Dividida em Esquerda e Direita) -->
                    <div class="grid grid-cols-1 md:grid-cols-3 gap-4 mb-6">
                        <!-- Esquerda: Histórico de Vendas -->
                        <div class="col-span-2 bg-white p-4 rounded shadow">
                            <h3 class="text-xl font-semibold mb-2">Histórico de Vendas</h3>
                            <div class="overflow-x-auto">
                                <table class="w-full rounded">
                                    <thead class="bg-gray-200">
                                        <tr>
                                            <th class="p-2 text-center">ID</th>
                                            <th class="p-2 text-center">Cliente</th>
                                            <th class="p-2 text-center">Produto</th>
                                            <th class="p-2 text-center">Quantidade</th>
                                            <th class="p-2 text-center">Data Entrega</th>
                                            <th class="p-2 text-center">Valor Total</th>
                                            <th class="p-2 text-center">Pago</th>
                                        </tr>
                                    </thead>
                                    <tbody>
                                        <?php
                                        try {
                                            $stmt = $db->prepare("SELECT v.id, c.nome, p.nome as produto, v.quantidade, v.data_entrega, v.valor_total, SUM(COALESCE(pg.valor, 0)) as pago 
                                                FROM vendas v 
                                                JOIN clientes c ON v.cliente_id = c.id 
                                                JOIN produtos p ON v.produto_id = p.id 
                                                LEFT JOIN pagamentos pg ON v.id = pg.venda_id 
                                                WHERE v.user_id = :user_id
                                                GROUP BY v.id, c.nome, p.nome, v.quantidade, v.data_entrega, v.valor_total");
                                            $stmt->execute([':user_id' => $user_id]);
                                            $vendas = $stmt->fetchAll(PDO::FETCH_ASSOC);
                                            foreach ($vendas as $venda) {
                                                echo "<tr>
                                                    <td class='p-2 text-center'>{$venda['id']}</td>
                                                    <td class='p-2 text-center'>{$venda['nome']}</td>
                                                    <td class='p-2 text-center'>{$venda['produto']}</td>
                                                    <td class='p-2 text-center'>{$venda['quantidade']}</td>
                                                    <td class='p-2 text-center'>{$venda['data_entrega']}</td>
                                                    <td class='p-2 text-center'>R$ " . number_format($venda['valor_total'], 2, ',', '.') . "</td>
                                                    <td class='p-2 text-center'>R$ " . number_format($venda['pago'], 2, ',', '.') . "</td>
                                                </tr>";
                                            }
                                        } catch (PDOException $e) {
                                            echo "<tr><td colspan='7' class='p-2 text-red-500 text-center'>Erro ao carregar vendas: " . $e->getMessage() . "</td></tr>";
                                            error_log("Erro ao carregar vendas em index.php: " . $e->getMessage());
                                        }
                                        ?>
                                    </tbody>
                                </table>
                            </div>
                        </div>

                        <!-- Direita: Catálogo de Produtos -->
                        <div class="bg-white p-4 rounded shadow">
                            <h3 class="text-xl font-semibold mb-2">Catálogo de Produtos</h3>
                            <form method="POST" enctype="multipart/form-data" class="mb-4">
    <input type="hidden" name="add_produto" value="1">
    <div class="mb-2">
        <label class="block text-gray-700">Nome</label>
        <input type="text" name="nome" required class="w-full p-2 border rounded">
    </div>
    <div class="mb-2">
        <label class="block text-gray-700">Preço de Compra (R$)</label>
        <input type="number" step="0.01" name="preco" required class="w-full p-2 border rounded">
    </div>
    <div class="mb-2">
        <label class="block text-gray-700">Preço de Venda (R$)</label>
        <input type="number" step="0.01" name="preco_venda" required class="w-full p-2 border rounded">
    </div>
    <div class="mb-2">
        <label class="block text-gray-700">Estoque</label>
        <input type="number" name="estoque" required class="w-full p-2 border rounded">
    </div>
    <div class="mb-2">
        <label class="block text-gray-700">Foto do Produto</label>
        <input type="file" name="foto" accept="image/*" class="w-full p-2 border rounded">
    </div>
    <button type="submit" class="bg-blue-600 text-white p-2 rounded hover:bg-blue-700 transition duration-300 hover:scale-105">Adicionar</button>
</form>
                            <div class="overflow-y-auto max-h-64">
                                <?php
                                try {
                                    $stmt = $db->prepare("SELECT * FROM produtos WHERE user_id = :user_id");
$stmt->execute([':user_id' => $user_id]);
$produtos = $stmt->fetchAll(PDO::FETCH_ASSOC);

// Depuração: Verificar se os produtos estão sendo retornados
if (!$produtos) {
    echo "<div class='p-2 text-red-500'>Nenhum produto encontrado para este usuário (user_id: $user_id).</div>";
    error_log("Nenhum produto encontrado para user_id: $user_id no dashboard em index.php");
} else {
    foreach ($produtos as $produto) {
        $foto = $produto['foto'] ? "<img src='{$produto['foto']}' alt='{$produto['nome']}' class='w-8 h-8 inline-block mr-2 object-cover'>" : "[Sem foto]";
        $preco_venda = isset($produto['preco_venda']) ? number_format($produto['preco_venda'], 2, ',', '.') : number_format($produto['preco'], 2, ',', '.');
        echo "<div class='p-2 border-b'>{$foto} <strong>{$produto['nome']}</strong> - R$ {$preco_venda} (Estoque: {$produto['estoque']})</div>";
    }
}
                                ?>
                            </div>
                        </div>
                    </div>

                    <!-- Seção Inferior: Registro de Vendas e Pagamentos -->
                    <div class="bg-white p-4 rounded shadow">
                        <h3 class="text-xl font-semibold mb-2">Registro de Vendas e Pagamentos</h3>
                        <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                            <!-- Registro de Vendas -->
                            <div>
                                <h4 class="text-lg font-semibold mb-2">Registrar Venda</h4>
                                <form method="POST">
                                    <input type="hidden" name="add_venda" value="1">
                                    <div class="mb-2">
                                        <label class="block text-gray-700">Cliente</label>
                                        <select name="cliente_id" required class="w-full p-2 border rounded">
                                            <?php
                                            try {
                                                $stmt = $db->prepare("SELECT * FROM clientes WHERE user_id = :user_id");
                                                $stmt->execute([':user_id' => $user_id]);
                                                $clientes = $stmt->fetchAll(PDO::FETCH_ASSOC);
                                                if (empty($clientes)) {
                                                    echo "<option value='' disabled selected>Nenhum cliente cadastrado</option>";
                                                } else {
                                                    foreach ($clientes as $cliente) {
                                                        echo "<option value='{$cliente['id']}'>{$cliente['nome']}</option>";
                                                    }
                                                }
                                            } catch (PDOException $e) {
                                                echo "<option value='' disabled selected>Erro ao carregar clientes: " . $e->getMessage() . "</option>";
                                                error_log("Erro ao carregar clientes em index.php: " . $e->getMessage());
                                            }
                                            ?>
                                        </select>
                                    </div>
                                    <div class="mb-2">
                                        <label class="block text-gray-700">Produto</label>
                                        <select name="produto_id" required class="w-full p-2 border rounded">
                                            <?php
                                            try {
                                                $stmt = $db->prepare("SELECT * FROM produtos WHERE user_id = :user_id");
                                                $stmt->execute([':user_id' => $user_id]);
                                                $produtos = $stmt->fetchAll(PDO::FETCH_ASSOC);
                                                if (empty($produtos)) {
                                                    echo "<option value='' disabled selected>Nenhum produto cadastrado</option>";
                                                } else {
                                                    foreach ($produtos as $produto) {
                                                        echo "<option value='{$produto['id']}'>{$produto['nome']} - R$ " . number_format($produto['preco'], 2, ',', '.') . "</option>";
                                                    }
                                                }
                                            } catch (PDOException $e) {
                                                echo "<option value='' disabled selected>Erro ao carregar produtos: " . $e->getMessage() . "</option>";
                                                error_log("Erro ao carregar produtos em index.php: " . $e->getMessage());
                                            }
                                            ?>
                                        </select>
                                    </div>
                                    <div class="mb-2">
                                        <label class="block text-gray-700">Quantidade</label>
                                        <input type="number" name="quantidade" required class="w-full p-2 border rounded" min="1">
                                    </div>
                                    <div class="mb-2">
                                        <label class="block text-gray-700">Data de Entrega</label>
                                        <input type="date" name="data_entrega" required class="w-full p-2 border rounded">
                                    </div>
                                    <div class="mb-2">
                                        <label class="block text-gray-700">Valor Total (R$)</label>
                                        <input type="number" step="0.01" name="valor_total" required class="w-full p-2 border rounded">
                                    </div>
                                    <button type="submit" class="bg-blue-600 text-white p-2 rounded hover:bg-blue-700 transition duration-300 hover:scale-105">Registrar Venda</button>
                                </form>
                            </div>

                            <!-- Lançamento de Pagamentos -->
                            <div>
                                <h4 class="text-lg font-semibold mb-2">Lançar Pagamento</h4>
                                <form method="POST">
                                    <input type="hidden" name="add_pagamento" value="1">
                                    <div class="mb-2">
                                        <label class="block text-gray-700">Venda</label>
                                        <select name="venda_id" required class="w-full p-2 border rounded">
                                            <?php
                                            try {
                                                $stmt = $db->prepare("SELECT v.id, c.nome, p.nome as produto FROM vendas v JOIN clientes c ON v.cliente_id = c.id JOIN produtos p ON v.produto_id = p.id WHERE v.user_id = :user_id");
                                                $stmt->execute([':user_id' => $user_id]);
                                                $vendas = $stmt->fetchAll(PDO::FETCH_ASSOC);
                                                if (empty($vendas)) {
                                                    echo "<option value='' disabled selected>Nenhuma venda registrada</option>";
                                                } else {
                                                    foreach ($vendas as $venda) {
                                                        echo "<option value='{$venda['id']}'>{$venda['nome']} - {$venda['produto']}</option>";
                                                    }
                                                }
                                            } catch (PDOException $e) {
                                                echo "<option value='' disabled selected>Erro ao carregar vendas: " . $e->getMessage() . "</option>";
                                                error_log("Erro ao carregar vendas em index.php: " . $e->getMessage());
                                            }
                                            ?>
                                        </select>
                                    </div>
                                    <div class="mb-2">
                                        <label class="block text-gray-700">Valor Pago (R$)</label>
                                        <input type="number" step="0.01" name="valor" required class="w-full p-2 border rounded">
                                    </div>
                                    <div class="mb-2">
                                        <label class="block text-gray-700">Data do Pagamento</label>
                                        <input type="date" name="data_pagamento" required class="w-full p-2 border rounded">
                                    </div>
                                    <button type="submit" class="bg-green-600 text-white p-2 rounded hover:bg-green-700 w-full">Lançar Pagamento</button>
                                </form>
                            </div>
                        </div>
                    </div>
                    <?php
                    break;

				case 'clientes':
                // Verificar se estamos visualizando o relatório de um cliente
                $relatorioCliente = null;
                if (isset($_GET['relatorio_cliente'])) {
                try {
                $stmt = $db->prepare("SELECT * FROM clientes WHERE id = :id AND user_id = :user_id");
                $stmt->execute([':id' => $_GET['relatorio_cliente'], ':user_id' => $user_id]);
                $relatorioCliente = $stmt->fetch(PDO::FETCH_ASSOC);
        } catch (PDOException $e) {
                $_SESSION['error'] = "Erro ao carregar cliente: " . $e->getMessage();
                error_log("Erro ao carregar cliente em index.php: " . $e->getMessage());
                header("Location: ?page=clientes");
            exit;
        }
    }

    if ($relatorioCliente) {
        ?>
        <h2 class="text-2xl font-bold mb-4">Relatório do Cliente: <?php echo htmlspecialchars($relatorioCliente['nome']); ?></h2>
        <div class="bg-white p-6 rounded shadow-md mb-6">
            <h3 class="text-xl font-semibold mb-2">Histórico de Compras</h3>
            <div class="overflow-x-auto">
                <table class="w-full rounded">
                    <thead class="bg-gray-200">
                        <tr>
                            <th class="p-2 text-center">ID</th>
                            <th class="p-2 text-center">Data da Compra</th>
                            <th class="p-2 text-center">Produto</th>
                            <th class="p-2 text-center">Valor Total</th>
                            <th class="p-2 text-center">Pagamento Efetuado</th>
                            <th class="p-2 text-center">Pagamento Restante</th>
                        </tr>
                    </thead>
                    <tbody>
                        <?php
                        try {
                            $stmt = $db->prepare("SELECT v.id, v.data_entrega, p.nome as produto, v.valor_total, SUM(COALESCE(pg.valor, 0)) as pago 
                                FROM vendas v 
                                JOIN produtos p ON v.produto_id = p.id 
                                LEFT JOIN pagamentos pg ON v.id = pg.venda_id 
                                WHERE v.cliente_id = :cliente_id AND v.user_id = :user_id
                                GROUP BY v.id, v.data_entrega, p.nome, v.valor_total");
                            $stmt->execute([':cliente_id' => $relatorioCliente['id'], ':user_id' => $user_id]);
                            $compras = $stmt->fetchAll(PDO::FETCH_ASSOC);
                            if (empty($compras)) {
                                echo "<tr><td colspan='6' class='p-2 text-center'>Nenhuma compra registrada</td></tr>";
                            } else {
                                foreach ($compras as $compra) {
                                    $restante = $compra['valor_total'] - $compra['pago'];
                                    echo "<tr class='table-center'>
                                        <td class='p-2 text-center'>{$compra['id']}</td>
                                        <td class='p-2 text-center'>{$compra['data_entrega']}</td>
                                        <td class='p-2 text-center'>{$compra['produto']}</td>
                                        <td class='p-2 text-center'>R$ " . number_format($compra['valor_total'], 2, ',', '.') . "</td>
                                        <td class='p-2 text-center'>R$ " . number_format($compra['pago'], 2, ',', '.') . "</td>
                                        <td class='p-2 text-center'>R$ " . number_format($restante, 2, ',', '.') . "</td>
                                    </tr>";
                                }
                            }
                        } catch (PDOException $e) {
                            echo "<tr><td colspan='6' class='p-2 text-red-500'>Erro ao carregar compras: " . $e->getMessage() . "</td></tr>";
                            error_log("Erro ao carregar compras em index.php: " . $e->getMessage());
                        }
                        ?>
                    </tbody>
                </table>
            </div>
            <a href="?page=clientes" class="mt-4 inline-block bg-gray-500 text-white p-2 rounded hover:bg-gray-600">Voltar</a>
        </div>
        <?php
    } else {
        // Verificar se estamos editando um cliente
        $editCliente = null;
        if (isset($_GET['edit_cliente'])) {
            try {
                $stmt = $db->prepare("SELECT * FROM clientes WHERE id = :id AND user_id = :user_id");
                $stmt->execute([':id' => $_GET['edit_cliente'], ':user_id' => $user_id]);
                $editCliente = $stmt->fetch(PDO::FETCH_ASSOC);
            } catch (PDOException $e) {
                $_SESSION['error'] = "Erro ao carregar cliente: " . $e->getMessage();
                error_log("Erro ao carregar cliente em index.php: " . $e->getMessage());
                header("Location: ?page=clientes");
                exit;
            }
        }
        ?>
        <!-- Cadastro de Clientes -->
        <h2 class="text-2xl font-bold mb-4"><?php echo $editCliente ? 'Editar Cliente' : 'Cadastro de Clientes'; ?></h2>
        <div class="bg-white p-6 rounded shadow-md mb-6">
            <form method="POST">
                <input type="hidden" name="<?php echo $editCliente ? 'edit_cliente' : 'add_cliente'; ?>" value="1">
                <?php if ($editCliente): ?>
                    <input type="hidden" name="id" value="<?php echo $editCliente['id']; ?>">
                <?php endif; ?>
                <div class="mb-4">
                    <label class="block text-gray-700">Nome</label>
                    <input type="text" name="nome" required class="w-full p-2 border rounded" value="<?php echo $editCliente ? htmlspecialchars($editCliente['nome']) : ''; ?>">
                </div>
                <div class="mb-4">
                    <label class="block text-gray-700">Telefone</label>
                    <input type="text" name="telefone" class="w-full p-2 border rounded" value="<?php echo $editCliente ? htmlspecialchars($editCliente['telefone']) : ''; ?>">
                </div>
                <div class="mb-4">
                    <label class="block text-gray-700">Endereço</label>
                    <input type="text" name="endereco" required class="w-full p-2 border rounded" placeholder="Rua, Avenida, etc." value="<?php echo $editCliente ? htmlspecialchars($editCliente['endereco']) : ''; ?>">
                </div>
                <div class="mb-4">
                    <label class="block text-gray-700">Número</label>
                    <input type="text" name="numero" required class="w-full p-2 border rounded" value="<?php echo $editCliente ? htmlspecialchars($editCliente['numero']) : ''; ?>">
                </div>
                <div class="mb-4">
                    <label class="block text-gray-700">Bairro</label>
                    <input type="text" name="bairro" required class="w-full p-2 border rounded" value="<?php echo $editCliente ? htmlspecialchars($editCliente['bairro']) : ''; ?>">
                </div>
                <div class="mb-4">
                    <label class="block text-gray-700">Cidade</label>
                    <input type="text" name="cidade" required class="w-full p-2 border rounded" value="<?php echo $editCliente ? htmlspecialchars($editCliente['cidade']) : ''; ?>">
                </div>
                <button type="submit" class="bg-blue-600 text-white p-2 rounded hover:bg-blue-700 transition duration-300 hover:scale-105"><?php echo $editCliente ? 'Atualizar' : 'Cadastrar'; ?></button>
                <?php if ($editCliente): ?>
                    <a href="?page=clientes" class="ml-2 bg-gray-500 text-white p-2 rounded hover:bg-gray-600">Cancelar</a>
                <?php endif; ?>
            </form>
        </div>

        <!-- Lista de Clientes -->
        <h3 class="text-xl font-semibold mb-2">Lista de Clientes</h3>
        <div class="overflow-x-auto">
            <table class="w-full bg-white rounded shadow">
                <thead class="bg-gray-200">
                    <tr>
                        <th class="p-2 text-center">ID</th>
                        <th class="p-2 text-center">Nome</th>
                        <th class="p-2 text-center">Telefone</th>
                        <th class="p-2 text-center">Endereço</th>
                        <th class="p-2 text-center">Número</th>
                        <th class="p-2 text-center">Bairro</th>
                        <th class="p-2 text-center">Cidade</th>
                        <th class="p-2 text-center">Ações</th>
                    </tr>
                </thead>
                <tbody>
                    <?php
                    try {
                        $stmt = $db->prepare("SELECT * FROM clientes WHERE user_id = :user_id");
                        $stmt->execute([':user_id' => $user_id]);
                        $clientes = $stmt->fetchAll(PDO::FETCH_ASSOC);
                        if (empty($clientes)) {
                            echo "<tr><td colspan='8' class='p-2 text-center'>Nenhum cliente cadastrado</td></tr>";
                        } else {
                            foreach ($clientes as $cliente) {
                                echo "<tr class='table-center'>
                                    <td class='p-2 text-center'>{$cliente['id']}</td>
                                    <td class='p-2 text-center'>{$cliente['nome']}</td>
                                    <td class='p-2 text-center'>{$cliente['telefone']}</td>
                                    <td class='p-2 text-center'>{$cliente['endereco']}</td>
                                    <td class='p-2 text-center'>{$cliente['numero']}</td>
                                    <td class='p-2 text-center'>{$cliente['bairro']}</td>
                                    <td class='p-2 text-center'>{$cliente['cidade']}</td>
                                    <td class='p-2 text-center'>
                                        <a href='?page=clientes&edit_cliente={$cliente['id']}' class='bg-yellow-500 text-white px-2 py-1 rounded hover:bg-yellow-600'>Editar</a>
                                        <a href='?page=clientes&relatorio_cliente={$cliente['id']}' class='bg-blue-500 text-white px-2 py-1 rounded hover:bg-blue-600 ml-2'>Relatório</a>
                                        <form method='POST' style='display:inline;' onsubmit='return confirm(\"Tem certeza que deseja excluir o cliente {$cliente['nome']}?\");'>
                                            <input type='hidden' name='delete_cliente' value='1'>
                                            <input type='hidden' name='id' value='{$cliente['id']}'>
                                            <button type='submit' class='bg-red-500 text-white px-2 py-1 rounded hover:bg-red-600'>Excluir</button>
                                        </form>
                                    </td>
                                </tr>";
                            }
                        }
                    } catch (PDOException $e) {
                        echo "<tr><td colspan='8' class='p-2 text-red-500'>Erro ao carregar clientes: " . $e->getMessage() . "</td></tr>";
                        error_log("Erro ao carregar clientes em index.php: " . $e->getMessage());
                    }
                    ?>
                </tbody>
            </table>
        </div>
        <?php
    }
    break;
                case 'produtos':
                    // Verificar se estamos editando um produto
                    $editProduto = null;
                    if (isset($_GET['edit_produto'])) {
                        try {
                            $stmt = $db->prepare("SELECT * FROM produtos WHERE id = :id AND user_id = :user_id");
                            $stmt->execute([':id' => $_GET['edit_produto'], ':user_id' => $user_id]);
                            $editProduto = $stmt->fetch(PDO::FETCH_ASSOC);
                        } catch (PDOException $e) {
                            $_SESSION['error'] = "Erro ao carregar produto: " . $e->getMessage();
                            error_log("Erro ao carregar produto em index.php: " . $e->getMessage());
                            header("Location: ?page=produtos");
                            exit;
                        }
                    }
                    ?>
                    <!-- Catálogo de Produtos -->
                    <h2 class="text-2xl font-bold mb-4"><?php echo $editProduto ? 'Editar Produto' : 'Catálogo de Produtos'; ?></h2>
                    <div class="bg-white p-6 rounded shadow-md mb-6">
                        <form method="POST" enctype="multipart/form-data">
                            <input type="hidden" name="<?php echo $editProduto ? 'edit_produto' : 'add_produto'; ?>" value="1">
                            <?php if ($editProduto): ?>
                                <input type="hidden" name="id" value="<?php echo $editProduto['id']; ?>">
                                <input type="hidden" name="foto_atual" value="<?php echo $editProduto['foto']; ?>">
                            <?php endif; ?>
                            <div class="mb-4">
                                <label class="block text-gray-700">Nome do Produto</label>
                                <input type="text" name="nome" required class="w-full p-2 border rounded" value="<?php echo $editProduto ? htmlspecialchars($editProduto['nome']) : ''; ?>">
                            </div>
                            <div class="mb-4">
    <label class="block text-gray-700">Preço de Compra (R$)</label>
    <input type="number" step="0.01" name="preco" required class="w-full p-2 border rounded" value="<?php echo $editProduto ? htmlspecialchars($editProduto['preco']) : ''; ?>">
</div>
<div class="mb-4">
    <label class="block text-gray-700">Preço de Venda (R$)</label>
    <input type="number" step="0.01" name="preco_venda" required class="w-full p-2 border rounded" value="<?php echo $editProduto && isset($editProduto['preco_venda']) ? htmlspecialchars($editProduto['preco_venda']) : ''; ?>">
</div>
                            <div class="mb-4">
                                <label class="block text-gray-700">Estoque</label>
                                <input type="number" name="estoque" required class="w-full p-2 border rounded" value="<?php echo $editProduto ? htmlspecialchars($editProduto['estoque']) : ''; ?>">
                            </div>
                            <div class="mb-4">
                                <label class="block text-gray-700">Foto do Produto</label>
                                <?php if ($editProduto && $editProduto['foto']): ?>
                                    <div class="mb-2">
                                        <img src="<?php echo $editProduto['foto']; ?>" alt="<?php echo htmlspecialchars($editProduto['nome']); ?>" class="w-16 h-16 object-cover">
                                        <p class="text-sm text-gray-600">Foto atual. Selecione uma nova para substituir.</p>
                                    </div>
                                <?php endif; ?>
                                <input type="file" name="foto" accept="image/*" class="w-full p-2 border rounded">
                            </div>
                            <button type="submit" class="bg-blue-600 text-white p-2 rounded hover:bg-blue-700 transition duration-300 hover:scale-105"><?php echo $editProduto ? 'Atualizar' : 'Adicionar'; ?></button>
                            <?php if ($editProduto): ?>
                                <a href="?page=produtos" class="ml-2 bg-gray-500 text-white p-2 rounded hover:bg-gray-600">Cancelar</a>
                            <?php endif; ?>
                        </form>
                    </div>

                    <!-- Produtos Cadastrados -->
                    <h3 class="text-xl font-semibold mb-2">Produtos Cadastrados</h3>
                    <div class="overflow-x-auto">
                        	                        <table class="w-full bg-white rounded shadow">
	                            <thead class="bg-gray-200">
	                                <tr>
	                                    <th class="p-2 text-center">ID</th>
	                                    <th class="p-2 text-center">Foto</th>
	                                    <th class="p-2 text-center">Nome</th>
	                                    <th class="p-2 text-center">Preço</th>
	                                    <th class="p-2 text-center">Estoque</th>
	                                    <th class="p-2 text-center">Ações</th>
	                                </tr>
	                            </thead>
	                            <tbody>
	                                <?php
	                                try {
	                                    $stmt = $db->prepare("SELECT * FROM produtos WHERE user_id = :user_id");
$stmt->execute([':user_id' => $user_id]);
$produtos = $stmt->fetchAll(PDO::FETCH_ASSOC);

// Depuração: Verificar se os produtos estão sendo retornados
if (!$produtos) {
    echo "<div class='p-2 text-red-500'>Nenhum produto encontrado para este usuário (user_id: $user_id).</div>";
    error_log("Nenhum produto encontrado para user_id: $user_id em index.php");
} else {
    foreach ($produtos as $produto) {
        $foto = $produto['foto'] ? "<img src='{$produto['foto']}' alt='{$produto['nome']}' class='w-16 h-16 object-cover'>" : "Sem foto";
        $preco_venda = isset($produto['preco_venda']) ? number_format($produto['preco_venda'], 2, ',', '.') : number_format($produto['preco'], 2, ',', '.');
        echo "<tr>
            <td class='p-2 text-center'>{$produto['id']}</td>
            <td class='p-2 text-center'>{$foto}</td>
            <td class='p-2 text-center'>{$produto['nome']}</td>
            <td class='p-2 text-center'>R$ {$preco_venda}</td>
            <td class='p-2 text-center'>{$produto['estoque']}</td>
            <td class='p-2 text-center'>
                <a href='?page=produtos&edit_produto={$produto['id']}' class='bg-yellow-500 text-white px-2 py-1 rounded hover:bg-yellow-600'>Editar</a>
                <form method='POST' style='display:inline;' onsubmit='return confirm(\"Tem certeza que deseja excluir o produto {$produto['nome']}?\");'>
                    <input type='hidden' name='delete_produto' value='1'>
                    <input type='hidden' name='id' value='{$produto['id']}'>
                    <button type='submit' class='bg-red-500 text-white px-2 py-1 rounded hover:bg-red-600'>Excluir</button>
                </form>
            </td>
        </tr>";
    }
}
	                                } catch (PDOException $e) {
	                                    echo "<tr><td colspan='6' class='p-2 text-red-500'>Erro ao carregar produtos: " . $e->getMessage() . "</td></tr>";
	                                    error_log("Erro ao carregar produtos em index.php: " . $e->getMessage());
	                                }
	                                ?>
	                            </tbody>
	                        </table>
                    </div>
                    <?php
                    break;

                case 'vendas':
                    ?>
					<script>
function updateValorVenda() {
    const produtoSelect = document.getElementById('produto_id');
    const quantidadeInput = document.getElementById('quantidade');
    const descontoInput = document.getElementById('desconto');
    const valorTotalInput = document.getElementById('valor_total');

    const selectedOption = produtoSelect.options[produtoSelect.selectedIndex];
    const precoVenda = parseFloat(selectedOption ? selectedOption.getAttribute('data-preco-venda') : 0);
    const quantidade = parseInt(quantidadeInput.value) || 1;
    const desconto = parseFloat(descontoInput.value) || 0;

    let valorTotal = precoVenda * quantidade;
    if (desconto > 0) {
        valorTotal -= (valorTotal * desconto / 100);
    }

    valorTotalInput.value = valorTotal.toFixed(2);
}
</script>					
                    <!-- Registro de Vendas -->
                    <h2 class="text-2xl font-bold mb-4">Registro de Vendas</h2>
                    <div class="bg-white p-6 rounded shadow-md mb-6">
                        <form method="POST">
                            <input type="hidden" name="add_venda" value="1">
                            <div class="mb-4">
                                <label class="block text-gray-700">Cliente</label>
                                <select name="cliente_id" required class="w-full p-2 border rounded">
                                    <?php
                                    try {
                                        $stmt = $db->prepare("SELECT * FROM clientes WHERE user_id = :user_id");
                                        $stmt->execute([':user_id' => $user_id]);
                                        $clientes = $stmt->fetchAll(PDO::FETCH_ASSOC);
                                        if (empty($clientes)) {
                                            echo "<option value='' disabled selected>Nenhum cliente cadastrado</option>";
                                        } else {
                                            foreach ($clientes as $cliente) {
                                                echo "<option value='{$cliente['id']}'>{$cliente['nome']}</option>";
                                            }
                                        }
                                    } catch (PDOException $e) {
                                        echo "<option value='' disabled selected>Erro ao carregar clientes: " . $e->getMessage() . "</option>";
                                        error_log("Erro ao carregar clientes em index.php: " . $e->getMessage());
                                    }
                                    ?>
                                </select>
                            </div>
                            <div class="mb-4">
    <label class="block text-gray-700">Produto</label>
    <select name="produto_id" id="produto_id" required class="w-full p-2 border rounded" onchange="updateValorVenda()">
        <?php
        try {
            $stmt = $db->prepare("SELECT * FROM produtos WHERE user_id = :user_id");
            $stmt->execute([':user_id' => $user_id]);
            $produtos = $stmt->fetchAll(PDO::FETCH_ASSOC);
            if (empty($produtos)) {
                echo "<option value='' disabled selected>Nenhum produto cadastrado</option>";
            } else {
                foreach ($produtos as $produto) {
                    $preco_venda = isset($produto['preco_venda']) ? $produto['preco_venda'] : $produto['preco'];
                    echo "<option value='{$produto['id']}' data-preco-venda='{$preco_venda}'>{$produto['nome']} - R$ " . number_format($preco_venda, 2, ',', '.') . "</option>";
                }
            }
        } catch (PDOException $e) {
            echo "<option value='' disabled selected>Erro ao carregar produtos: " . $e->getMessage() . "</option>";
            error_log("Erro ao carregar produtos em index.php: " . $e->getMessage());
        }
        ?>
    </select>
</div>
<div class="mb-4">
    <label class="block text-gray-700">Quantidade</label>
    <input type="number" name="quantidade" id="quantidade" required class="w-full p-2 border rounded" min="1" oninput="updateValorVenda()">
</div>
<div class="mb-4">
    <label class="block text-gray-700">Forma de Pagamento</label>
    <select name="forma_pagamento" id="forma_pagamento" required class="w-full p-2 border rounded">
        <option value="Pix">Pix</option>
        <option value="Dinheiro">Dinheiro</option>
        <option value="Débito">Débito</option>
        <option value="Cartão de Crédito">Cartão de Crédito</option>
    </select>
</div>
<div class="mb-4">
    <label class="block text-gray-700">Desconto (%)</label>
    <input type="number" name="desconto" id="desconto" step="0.01" min="0" max="100" class="w-full p-2 border rounded" placeholder="0" oninput="updateValorVenda()">
</div>
<div class="mb-4">
    <label class="block text-gray-700">Data de Entrega</label>
    <input type="date" name="data_entrega" required class="w-full p-2 border rounded">
</div>
<div class="mb-4">
    <label class="block text-gray-700">Valor da Venda R$</label>
    <input type="number" step="0.01" name="valor_total" id="valor_total" required class="w-full p-2 border rounded" readonly>
</div>
                            <button type="submit" class="bg-blue-600 text-white p-2 rounded hover:bg-blue-700 transition duration-300 hover:scale-105">Registrar Venda</button>
                        </form>
                    </div>

                    <!-- Lançamento de Pagamentos -->
                    <h3 class="text-xl font-semibold mb-2">Lançamento de Pagamentos</h3>
                    <div class="bg-white p-6 rounded shadow-md mb-6">
                        <form method="POST">
                            <input type="hidden" name="add_pagamento" value="1">
                            <div class="mb-4">
                                <label class="block text-gray-700">Venda</label>
                                <select name="venda_id" required class="w-full p-2 border rounded">
                                    <?php
                                    try {
                                        $stmt = $db->prepare("SELECT v.id, c.nome, p.nome as produto FROM vendas v JOIN clientes c ON v.cliente_id = c.id JOIN produtos p ON v.produto_id = p.id WHERE v.user_id = :user_id");
                                        $stmt->execute([':user_id' => $user_id]);
                                        $vendas = $stmt->fetchAll(PDO::FETCH_ASSOC);
                                        if (empty($vendas)) {
                                            echo "<option value='' disabled selected>Nenhuma venda registrada</option>";
                                        } else {
                                            foreach ($vendas as $venda) {
                                                echo "<option value='{$venda['id']}'>{$venda['nome']} - {$venda['produto']}</option>";
                                            }
                                        }
                                    } catch (PDOException $e) {
                                        echo "<option value='' disabled selected>Erro ao carregar vendas: " . $e->getMessage() . "</option>";
                                        error_log("Erro ao carregar vendas em index.php: " . $e->getMessage());
                                    }
                                    ?>
                                </select>
                            </div>
                            <div class="mb-4">
                                <label class="block text-gray-700">Valor Pago (R$)</label>
                                <input type="number" step="0.01" name="valor" required class="w-full p-2 border rounded">
                            </div>
                            <div class="mb-4">
                                <label class="block text-gray-700">Data do Pagamento</label>
                                <input type="date" name="data_pagamento" required class="w-full p-2 border rounded">
                            </div>
                            <button type="submit" class="bg-green-600 text-white p-2 rounded hover:bg-green-700">Lançar Pagamento</button>
                        </form>
                    </div>

                    <!-- Histórico de Vendas -->
                    <h3 class="text-xl font-semibold mb-2">Histórico de Vendas</h3>
                    <div class="overflow-x-auto">
                        <table class="w-full bg-white rounded shadow">
                            <thead class="bg-gray-200">
                                <tr>
                                    <th class="p-2 text-center">ID</th>
                                    <th class="p-2 text-center">Cliente</th>
                                    <th class="p-2 text-center">Produto</th>
                                    <th class="p-2 text-center">Quantidade</th>
                                    <th class="p-2 text-center">Data Entrega</th>
                                    <th class="p-2 text-center">Valor Total</th>
                                    <th class="p-2 text-center">Pago</th>
                                </tr>
                            </thead>
                            <tbody>
                                <?php
                                try {
                                    $stmt = $db->prepare("SELECT v.id, c.nome, p.nome as produto, v.quantidade, v.data_entrega, v.valor_total, SUM(COALESCE(pg.valor, 0)) as pago 
                                        FROM vendas v 
                                        JOIN clientes c ON v.cliente_id = c.id 
                                        JOIN produtos p ON v.produto_id = p.id 
                                        LEFT JOIN pagamentos pg ON v.id = pg.venda_id 
                                        WHERE v.user_id = :user_id
                                        GROUP BY v.id, c.nome, p.nome, v.quantidade, v.data_entrega, v.valor_total");
                                    $stmt->execute([':user_id' => $user_id]);
                                    $vendas = $stmt->fetchAll(PDO::FETCH_ASSOC);
                                    if (empty($vendas)) {
                                        echo "<tr><td colspan='7' class='p-2 text-center'>Nenhuma venda registrada</td></tr>";
                                    } else {
                                        foreach ($vendas as $venda) {
                                            echo "<tr>
                                                <td class='p-2 text-center'>{$venda['id']}</td>
                                                <td class='p-2 text-center'>{$venda['nome']}</td>
                                                <td class='p-2 text-center'>{$venda['produto']}</td>
                                                <td class='p-2 text-center'>{$venda['quantidade']}</td>
                                                <td class='p-2 text-center'>{$venda['data_entrega']}</td>
                                                <td class='p-2 text-center'>R$ " . number_format($venda['valor_total'], 2, ',', '.') . "</td>
                                                <td class='p-2 text-center'>R$ " . number_format($venda['pago'], 2, ',', '.') . "</td>
                                            </tr>";
                                        }
                                    }
                                } catch (PDOException $e) {
                                    echo "<tr><td colspan='7' class='p-2 text-red-500 text-center'>Erro ao carregar vendas: " . $e->getMessage() . "</td></tr>";
                                    error_log("Erro ao carregar vendas em index.php: " . $e->getMessage());
                                }
                                ?>
                            </tbody>
                        </table>
                    </div>
                    <?php
                    break;

                case 'admin':
                    // Restringir acesso apenas ao usuário "thyago"
                    if ($_SESSION['username'] !== 'thyago') {
                        header("Location: ?page=dashboard");
                        exit;
                    }
                    ?>
                    <!-- Administração de Usuários -->
                    <h2 class="text-2xl font-bold mb-4">Administração de Usuários</h2>
                    <div class="overflow-x-auto">
                        <table class="w-full bg-white rounded shadow">
                            <thead class="bg-gray-200">
                                <tr>
                                    <th class="p-2">Nome</th>
                                    <th class="p-2">Login</th>
                                    <th class="p-2">E-mail</th>
                                    <th class="p-2">Ações</th>
                                </tr>
                            </thead>
                            <tbody>
                                <?php
                                try {
                                    $stmt = $db->prepare("SELECT * FROM users WHERE username != 'thyago'");
                                    $stmt->execute();
                                    $users = $stmt->fetchAll(PDO::FETCH_ASSOC);
                                    if (empty($users)) {
                                        echo "<tr><td colspan='4' class='p-2 text-center'>Nenhum usuário cadastrado</td></tr>";
                                    } else {
                                        foreach ($users as $user) {
                                            echo "<tr>
    <td class='p-2 text-center'>" . htmlspecialchars($user['full_name']) . "</td>
    <td class='p-2 text-center'>" . htmlspecialchars($user['username']) . "</td>
    <td class='p-2 text-center'>" . htmlspecialchars($user['email']) . "</td>
    <td class='p-2 text-center'>
        <form method='POST' style='display:inline;' onsubmit='return confirm(\"Tem certeza que deseja excluir o usuário " . htmlspecialchars($user['full_name']) . "?\");'>
            <input type='hidden' name='delete_user' value='1'>
            <input type='hidden' name='id' value='{$user['id']}'>
            <button type='submit' class='bg-red-500 text-white px-2 py-1 rounded hover:bg-red-600'>Excluir</button>
        </form>
    </td>
</tr>";
                                        }
                                    }
                                } catch (PDOException $e) {
                                    echo "<tr><td colspan='4' class='p-2 text-red-500'>Erro ao carregar usuários: " . $e->getMessage() . "</td></tr>";
                                    error_log("Erro ao carregar usuários em index.php: " . $e->getMessage());
                                }
                                ?>
                            </tbody>
                        </table>
                    </div>
                    <?php
                    break;

                case 'settings':
                    // Restringir acesso a usuários que não sejam "thyago"
                    if ($_SESSION['username'] === 'thyago') {
                        header("Location: ?page=dashboard");
                        exit;
                    }
                    ?>
                    <!-- Configurações do Usuário -->
                    <h2 class="text-2xl font-bold mb-4">Configurações</h2>
                    <div class="bg-white p-6 rounded shadow-md mb-6">
                        <h3 class="text-xl font-semibold mb-4">Alterar Nome do Seu Sistema</h3>
                        <form method="POST">
                            <input type="hidden" name="update_system_name" value="1">
                            <div class="mb-4">
                                <label for="system_name" class="block text-gray-700 mb-2">Nome do Sistema</label>
                                <input type="text" id="system_name" name="system_name" required class="w-full p-2 border rounded focus:outline-none focus:ring-2 focus:ring-blue-600" value="<?php echo htmlspecialchars($system_name === "Polly - Tupperware" ? "" : $system_name); ?>" placeholder="Digite o novo nome do sistema">
                            </div>
                            <button type="submit" class="bg-blue-600 text-white p-2 rounded hover:bg-blue-700 transition duration-300 hover:scale-105">Salvar</button>
                        </form>
                    </div>

                    <!-- Alterar Foto de Perfil -->
                    <div class="bg-white p-6 rounded shadow-md mb-6">
                        <h3 class="text-xl font-semibold mb-4">Alterar Foto de Perfil</h3>
                        <form method="POST" enctype="multipart/form-data">
                            <input type="hidden" name="upload_profile_photo" value="1">
                            <div class="mb-4">
                                <label for="profile_photo" class="block text-gray-700 mb-2">Foto de Perfil</label>
                                <?php if ($profile_photo): ?>
                                    <div class="mb-2">
                                        <img src="<?php echo htmlspecialchars($profile_photo); ?>" alt="Foto de Perfil Atual" class="w-32 h-32 rounded-full object-cover">
                                        <p class="text-sm text-gray-600">Foto atual. Selecione uma nova para substituir.</p>
                                    </div>
                                <?php endif; ?>
                                <input type="file" id="profile_photo" name="profile_photo" accept="image/*" class="w-full p-2 border rounded">
                            </div>
                            <button type="submit" class="bg-blue-600 text-white p-2 rounded hover:bg-blue-700 transition duration-300 hover:scale-105">Alterar foto</button>
                            <?php if ($profile_photo): ?>
                                <form method="POST" style="display:inline;" class="ml-2">
                                    <input type="hidden" name="remove_profile_photo" value="1">
                                    <button type="submit" class="bg-red-500 text-white p-2 rounded hover:bg-red-600">Remover Foto</button>
                                </form>
                            <?php endif; ?>
                        </form>
                    </div>
                    <?php
                    break;
					
				              
        <?php
        if (isset($_SESSION['report_data'])) {
            $report_data = $_SESSION['report_data'];
            $tipo = $report_data['tipo'];
            $dia = $report_data['dia'];
            $mes = $report_data['mes'];
            $ano = $report_data['ano'];

            try {
                if ($tipo === 'diario' && $dia && $ano) {
                    $stmt = $db->prepare("SELECT COUNT(*) as total_vendas, SUM(v.quantidade) as total_quantidade, SUM(v.valor_total) as total_dinheiro, SUM(COALESCE(pg.valor, 0)) as total_recebido 
                        FROM vendas v 
                        LEFT JOIN pagamentos pg ON v.id = pg.venda_id 
                        WHERE v.user_id = :user_id AND v.data_entrega = :data");
                    $stmt->execute([':user_id' => $user_id, ':data' => $dia]);
                } elseif ($tipo === 'mensal' && $mes && $ano) {
                    $stmt = $db->prepare("SELECT COUNT(*) as total_vendas, SUM(v.quantidade) as total_quantidade, SUM(v.valor_total) as total_dinheiro, SUM(COALESCE(pg.valor, 0)) as total_recebido 
                        FROM vendas v 
                        LEFT JOIN pagamentos pg ON v.id = pg.venda_id 
                        WHERE v.user_id = :user_id AND strftime('%m', v.data_entrega) = :mes AND strftime('%Y', v.data_entrega) = :ano");
                    $stmt->execute([':user_id' => $user_id, ':mes' => sprintf("%02d", $mes), ':ano' => $ano]);
                } elseif ($tipo === 'anual' && $ano) {
                    $stmt = $db->prepare("SELECT COUNT(*) as total_vendas, SUM(v.quantidade) as total_quantidade, SUM(v.valor_total) as total_dinheiro, SUM(COALESCE(pg.valor, 0)) as total_recebido 
                        FROM vendas v 
                        LEFT JOIN pagamentos pg ON v.id = pg.venda_id 
                        WHERE v.user_id = :user_id AND strftime('%Y', v.data_entrega) = :ano");
                    $stmt->execute([':user_id' => $user_id, ':ano' => $ano]);
                } else {
                    throw new Exception("Parâmetros de relatório inválidos.");
                }

                $result = $stmt->fetch(PDO::FETCH_ASSOC);
                $total_vendas = $result['total_vendas'] ?? 0;
                $total_quantidade = $result['total_quantidade'] ?? 0;
                $total_dinheiro = $result['total_dinheiro'] ?? 0;
                $total_recebido = $result['total_recebido'] ?? 0;
                $total_a_receber = $total_dinheiro - $total_recebido;

                // Dados para o gráfico
                $chart_labels = ['Vendas', 'Quantidade', 'Total (R$)', 'Recebido (R$)', 'A Receber (R$)'];
                $chart_data = [$total_vendas, $total_quantidade, $total_dinheiro, $total_recebido, $total_a_receber];
            } catch (Exception $e) {
                echo "<div class='text-red-500'>Erro ao gerar relatório: " . $e->getMessage() . "</div>";
                $total_vendas = $total_quantidade = $total_dinheiro = $total_recebido = $total_a_receber = 0;
                $chart_labels = [];
                $chart_data = [];
            }
            ?>
            <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
                <!-- Resumo do Relatório -->
                <div>
                    <h3 class="text-xl font-semibold mb-4">Resumo do Relatório</h3>
                    <div class="bg-gray-50 p-4 rounded shadow">
                        <p><strong>Tipo:</strong> <?php echo ucfirst($tipo); ?></p>
                        <?php if ($tipo === 'diario') echo "<p><strong>Data:</strong> " . date('d/m/Y', strtotime($dia)) . "</p>"; ?>
                        <?php if ($tipo === 'mensal') echo "<p><strong>Mês:</strong> " . date('F', mktime(0, 0, 0, $mes, 1)) . "/$ano</p>"; ?>
                        <?php if ($tipo === 'anual') echo "<p><strong>Ano:</strong> $ano</p>"; ?>
                        <p><strong>Total de Vendas:</strong> <?php echo $total_vendas; ?></p>
                        <p><strong>Quantidade Total:</strong> <?php echo $total_quantidade; ?></p>
                        <p><strong>Total Dinheiro:</strong> R$ <?php echo number_format($total_dinheiro, 2, ',', '.'); ?></p>
                        <p><strong>Total Recebido:</strong> R$ <?php echo number_format($total_recebido, 2, ',', '.'); ?></p>
                        <p><strong>Total a Receber:</strong> R$ <?php echo number_format($total_a_receber, 2, ',', '.'); ?></p>
                    </div>
                </div>
                <!-- Gráfico -->
                <div>
                    <h3 class="text-xl font-semibold mb-4">Visualização Gráfica</h3>
                    <canvas id="reportChart" class="w-full h-64"></canvas>
                </div>
            </div>
            <script>
                const ctx = document.getElementById('reportChart').getContext('2d');
                new Chart(ctx, {
                    type: 'bar',
                    data: {
                        labels: <?php echo json_encode($chart_labels); ?>,
                        datasets: [{
                            label: 'Relatório de Vendas',
                            data: <?php echo json_encode($chart_data); ?>,
                            backgroundColor: [
                                'rgba(59, 130, 246, 0.6)',  // Azul
                                'rgba(16, 185, 129, 0.6)',  // Verde
                                'rgba(234, 179, 8, 0.6)',   // Amarelo
                                'rgba(34, 197, 94, 0.6)',   // Verde claro
                                'rgba(239, 68, 68, 0.6)'    // Vermelho
                            ],
                            borderColor: [
                                'rgba(59, 130, 246, 1)',
                                'rgba(16, 185, 129, 1)',
                                'rgba(234, 179, 8, 1)',
                                'rgba(34, 197, 94, 1)',
                                'rgba(239, 68, 68, 1)'
                            ],
                            borderWidth: 1
                        }]
                    },
                    options: {
                        scales: {
                            y: {
                                beginAtZero: true
                            }
                        },
                        plugins: {
                            legend: {
                                display: true,
                                position: 'top'
                            }
                        }
                    }
                });

                function toggleFields() {
                    const reportType = document.getElementById('report_type').value;
                    const dayField = document.getElementById('day_field');
                    const monthField = document.getElementById('month_field');
                    const yearField = document.getElementById('year_field');

                    if (reportType === 'diario') {
                        dayField.style.display = 'block';
                        monthField.style.display = 'block';
                        yearField.style.display = 'block';
                    } else if (reportType === 'mensal') {
                        dayField.style.display = 'none';
                        monthField.style.display = 'block';
                        yearField.style.display = 'block';
                    } else if (reportType === 'anual') {
                        dayField.style.display = 'none';
                        monthField.style.display = 'none';
                        yearField.style.display = 'block';
                    }
                }

                // Inicializar visibilidade dos campos
                toggleFields();
            </script>
            <?php
            // Limpar os dados do relatório após exibição
            unset($_SESSION['report_data']);
        }
        ?>
    </div>
    <?php
    break;

    default:
        header("Location: ?page=dashboard");
        exit;
    }
    ?>
    </div>
</div>

<!-- Script para ocultar/mostrar a barra lateral -->
<script>
    const sidebar = document.getElementById('sidebar');
    const showSidebar = document.getElementById('showSidebar');
    const hideSidebar = document.getElementById('hideSidebar');
    const userMenuButton = document.getElementById('userMenuButton');
    const userMenu = document.getElementById('userMenu');

    // Evento para abrir/fechar o menu dropdown
    userMenuButton.addEventListener('click', () => {
        userMenu.classList.toggle('hidden');
    });

    // Fechar o menu dropdown ao clicar fora
    document.addEventListener('click', (e) => {
        if (!userMenuButton.contains(e.target) && !userMenu.contains(e.target)) {
            userMenu.classList.add('hidden');
        }
    });

    // Eventos para ocultar/mostrar a barra lateral
    hideSidebar.addEventListener('click', () => {
        sidebar.classList.add('hidden');
        showSidebar.classList.remove('hidden');
        mainContent.classList.remove('flex-1');
        mainContent.classList.add('w-full');
    });

    showSidebar.addEventListener('click', () => {
        sidebar.classList.remove('hidden');
        showSidebar.classList.add('hidden');
        mainContent.classList.remove('w-full');
        mainContent.classList.add('flex-1');
    });
</script>
</body>
</html>
