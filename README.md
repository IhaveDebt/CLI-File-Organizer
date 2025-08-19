// CLI File Organizer (C++17)
// Features:
// - Organize by file type (extension categories) or by last-write date (YYYY/MM).
// - Dry run (prints what would happen).
// - Undo using a manifest file written to the target directory.
// - Skips already-organized files and the manifest itself.
// Build: g++ -std=gnu++17 -O2 -o organize organize.cpp
// Usage examples:
//   ./organize --path "/path/to/folder" --by type
//   ./organize --path "/path/to/folder" --by date
//   ./organize --path "/path/to/folder" --by type --dry-run
//   ./organize --path "/path/to/folder" --undo

#include <algorithm>
#include <chrono>
#include <cctype>
#include <filesystem>
#include <fstream>
#include <iomanip>
#include <iostream>
#include <map>
#include <set>
#include <sstream>
#include <string>
#include <unordered_map>
#include <vector>

namespace fs = std::filesystem;

static const std::string kManifestName = ".organizer_manifest.csv";

struct Args {
    fs::path path;
    std::string by = "type"; // "type" or "date"
    bool dryRun = false;
    bool undo = false;
};

static void print_help(const char* argv0) {
    std::cout
      << "CLI File Organizer\n"
      << "Usage:\n"
      << "  " << argv0 << " --path <directory> [--by type|date] [--dry-run]\n"
      << "  " << argv0 << " --path <directory> --undo\n\n"
      << "Options:\n"
      << "  --path <dir>   Target directory to organize.\n"
      << "  --by <mode>    'type' (by extension category, default) or 'date' (YYYY/MM).\n"
      << "  --dry-run      Show actions without moving files.\n"
      << "  --undo         Revert last organization using manifest.\n";
}

static bool parse_args(int argc, char** argv, Args& a) {
    if (argc < 3) { print_help(argv[0]); return false; }
    for (int i = 1; i < argc; ++i) {
        std::string s = argv[i];
        if (s == "--path" && i + 1 < argc) {
            a.path = fs::u8path(argv[++i]);
        } else if (s == "--by" && i + 1 < argc) {
            a.by = argv[++i];
            std::transform(a.by.begin(), a.by.end(), a.by.begin(), [](unsigned char c){return std::tolower(c);});
        } else if (s == "--dry-run") {
            a.dryRun = true;
        } else if (s == "--undo") {
            a.undo = true;
        } else if (s == "--help" || s == "-h") {
            print_help(argv[0]);
            return false;
        } else {
            std::cerr << "Unknown argument: " << s << "\n";
            print_help(argv[0]);
            return false;
        }
    }
    if (a.path.empty()) {
        std::cerr << "--path is required\n";
        return false;
    }
    if (!fs::exists(a.path) || !fs::is_directory(a.path)) {
        std::cerr << "Path does not exist or is not a directory: " << a.path << "\n";
        return false;
    }
    if (!a.undo && a.by != "type" && a.by != "date") {
        std::cerr << "--by must be 'type' or 'date'\n";
        return false;
    }
    return true;
}

static std::string to_lower(std::string s) {
    std::transform(s.begin(), s.end(), s.begin(), [](unsigned char c){return std::tolower(c);});
    return s;
}

static std::string ext_of(const fs::path& p) {
    std::string e = to_lower(p.extension().string());
    if (!e.empty() && e[0]=='.') e.erase(0,1);
    return e;
}

static std::string category_for_ext(const std::string& ext) {
    static const std::unordered_map<std::string, std::string> table = {
        // images
        {"jpg","Images"},{"jpeg","Images"},{"png","Images"},{"gif","Images"},{"bmp","Images"},
        {"tiff","Images"},{"webp","Images"},{"heic","Images"},{"svg","Images"},
        // videos
        {"mp4","Videos"},{"mov","Videos"},{"mkv","Videos"},{"avi","Videos"},{"wmv","Videos"},
        {"flv","Videos"},{"webm","Videos"},
        // audio
        {"mp3","Audio"},{"wav","Audio"},{"flac","Audio"},{"aac","Audio"},{"ogg","Audio"},{"m4a","Audio"},
        // docs
        {"pdf","PDFs"},{"doc","Documents"},{"docx","Documents"},{"rtf","Documents"},{"txt","Documents"},
        {"odt","Documents"},{"md","Documents"},
        // spreadsheets
        {"xls","Spreadsheets"},{"xlsx","Spreadsheets"},{"csv","Spreadsheets"},{"ods","Spreadsheets"},
        // presentations
        {"ppt","Presentations"},{"pptx","Presentations"},{"odp","Presentations"},
        // archives
        {"zip","Archives"},{"rar","Archives"},{"7z","Archives"},{"tar","Archives"},{"gz","Archives"},
        // code
        {"c","Code"},{"h","Code"},{"hpp","Code"},{"cpp","Code"},{"cc","Code"},{"cs","Code"},
        {"java","Code"},{"py","Code"},{"js","Code"},{"ts","Code"},{"tsx","Code"},{"go","Code"},
        {"rs","Code"},{"php","Code"},{"rb","Code"},{"swift","Code"},{"kt","Code"},{"m","Code"},
        {"sh","Code"},{"bat","Code"},{"ps1","Code"},
        // executables / installers
        {"exe","Apps"},{"msi","Apps"},{"apk","Apps"},{"dmg","Apps"},{"pkg","Apps"},{"app","Apps"},
        // fonts
        {"ttf","Fonts"},{"otf","Fonts"},{"woff","Fonts"},{"woff2","Fonts"}
    };
    auto it = table.find(ext);
    return (it==table.end()) ? "Other" : it->second;
}

static std::string yyyymm_from_time(fs::file_time_type tp) {
    using namespace std::chrono;
    // Convert to system_clock for calendar conversion
    auto sctp = time_point_cast<system_clock::duration>(tp - fs::file_time_type::clock::now()
                + system_clock::now());
    std::time_t t = system_clock::to_time_t(sctp);
    std::tm tm{};
#ifdef _WIN32
    localtime_s(&tm, &t);
#else
    localtime_r(&t, &tm);
#endif
    std::ostringstream os;
    os << std::put_time(&tm, "%Y/%m");
    return os.str();
}

struct Move {
    fs::path from;
    fs::path to;
};

static void ensure_parent(const fs::path& p) {
    fs::path dir = p.parent_path();
    if (!dir.empty() && !fs::exists(dir)) fs::create_directories(dir);
}

static void write_manifest(const fs::path& root, const std::vector<Move>& moves) {
    fs::path manifest = root / kManifestName;
    std::ofstream out(manifest, std::ios::app);
    if (!out) {
        std::cerr << "Warning: cannot write manifest at " << manifest << "\n";
        return;
    }
    // Format: from_rel,to_rel
    for (const auto& m : moves) {
        fs::path fr = fs::relative(m.from, root);
        fs::path to = fs::relative(m.to, root);
        out << fr.generic_string() << "," << to.generic_string() << "\n";
    }
}

static std::vector<Move> load_manifest(const fs::path& root) {
    std::vector<Move> moves;
    fs::path manifest = root / kManifestName;
    std::ifstream in(manifest);
    if (!in) return moves;
    std::string line;
    while (std::getline(in, line)) {
        if (line.empty()) continue;
        auto comma = line.find(',');
        if (comma == std::string::npos) continue;
        fs::path fr = root / fs::path(line.substr(0, comma));
        fs::path to = root / fs::path(line.substr(comma + 1));
        moves.push_back({fr, to});
    }
    return moves;
}

static bool is_hidden_or_manifest(const fs::directory_entry& de, const fs::path& root) {
    const auto p = de.path();
    if (p.filename() == kManifestName) return true;
#ifdef _WIN32
    // No portable hidden flag; skip dotfiles below.
#endif
    std::string name = p.filename().string();
    if (!name.empty() && name[0] == '.') return true; // skip dotfiles/folders
    // Also skip our category folders to avoid reprocessing if user reruns.
    static const std::set<std::string> reserved = {
        "Images","Videos","Audio","PDFs","Documents","Spreadsheets","Presentations","Archives",
        "Code","Apps","Fonts","Other"
    };
    if (reserved.count(name)) return true;
    // Date folders like YYYY and MM: skip if exactly 4 digits or 2 digits in the right place
    if (p.parent_path() == root) {
        if (name.size() == 4 && std::all_of(name.begin(), name.end(), ::isdigit)) return true;
    }
    return false;
}

int main(int argc, char** argv) {
    std::ios::sync_with_stdio(false);
    Args args;
    if (!parse_args(argc, argv, args)) return 1;

    try {
        if (args.undo) {
            auto entries = load_manifest(args.path);
            if (entries.empty()) {
                std::cout << "Nothing to undo (no manifest or empty manifest).\n";
                return 0;
            }
            // Undo in reverse order for safety
            std::reverse(entries.begin(), entries.end());
            size_t ok = 0, fail = 0;
            for (const auto& mv : entries) {
                if (!fs::exists(mv.to)) { ++fail; continue; }
                if (args.dryRun) {
                    std::cout << "[UNDO-DRY] " << mv.to << " -> " << mv.from << "\n";
                    ++ok;
                } else {
                    ensure_parent(mv.from);
                    std::error_code ec;
                    fs::rename(mv.to, mv.from, ec);
                    if (ec) {
                        // fallback: try copy+remove if cross-device
                        std::error_code ec2;
                        fs::copy_file(mv.to, mv.from, fs::copy_options::overwrite_existing, ec2);
                        if (!ec2) fs::remove(mv.to);
                        (ec2 ? fail : ok) += 1;
                        if (ec2) std::cerr << "Failed to undo " << mv.to << " -> " << mv.from << ": " << ec2.message() << "\n";
                    } else {
                        ++ok;
                    }
                }
            }
            if (!args.dryRun) {
                // Clear manifest after successful undo attempt
                std::ofstream(args.path / kManifestName, std::ios::trunc).close();
            }
            std::cout << "Undo complete. OK=" << ok << " FAIL=" << fail << "\n";
            return 0;
        }

        // Organize mode
        std::vector<Move> moves;
        size_t scanned = 0, planned = 0;

        for (auto it = fs::directory_iterator(args.path); it != fs::directory_iterator(); ++it) {
            const auto& de = *it;
            if (!de.is_regular_file()) {
                // Optionally flatten subfolders: out of scope for simplicity.
                continue;
            }
            if (is_hidden_or_manifest(de, args.path)) continue;

            ++scanned;
            fs::path src = de.path();
            fs::path dst;
            if (args.by == "type") {
                std::string cat = category_for_ext(ext_of(src));
                dst = args.path / cat / src.filename();
            } else { // "date"
                auto tp = fs::last_write_time(src);
                auto yymm = yyyymm_from_time(tp); // "YYYY/MM"
                dst = args.path / yymm / src.filename();
            }

            if (fs::equivalent(src, dst)) continue; // already there
            moves.push_back({src, dst});
            ++planned;
        }

        if (moves.empty()) {
            std::cout << "No files to organize. Scanned=" << scanned << "\n";
            return 0;
        }

        for (const auto& m : moves) {
            if (args.dryRun) {
                std::cout << "[DRY] " << m.from << " -> " << m.to << "\n";
            } else {
                ensure_parent(m.to);
                std::error_code ec;
                fs::rename(m.from, m.to, ec);
                if (ec) {
                    // cross-device fallback
                    std::error_code ec2;
                    fs::copy_file(m.from, m.to, fs::copy_options::overwrite_existing, ec2);
                    if (!ec2) fs::remove(m.from);
                    if (ec2) {
                        std::cerr << "Failed: " << m.from << " -> " << m.to << " : " << ec2.message() << "\n";
                    }
                }
            }
        }

        if (!args.dryRun) write_manifest(args.path, moves);

        std::cout << (args.dryRun ? "Dry run complete. " : "Organize complete. ")
                  << "Scanned=" << scanned << " Planned=" << planned
                  << (args.by == "type" ? " Mode=type\n" : " Mode=date\n");

    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << "\n";
        return 2;
    }
    return 0;
}
