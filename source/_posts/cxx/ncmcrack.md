---
title: 音频转换工具
date: 2025/7/24 16:09:00 
categories:
- C/C++
tags:
- C/C++
---

### 相关代码

```c++
  // main.cc
  #include "utils.h"
  #include <boost/asio.hpp>
  #include <boost/asio/io_context.hpp>
  #include <boost/beast.hpp>
  #include <boost/beast/core/tcp_stream.hpp>
  #include <boost/beast/http/file_body.hpp>
  #include <boost/beast/http/message.hpp>
  #include <boost/beast/http/parser.hpp>
  #include <boost/beast/http/status.hpp>
  #include <boost/beast/http/string_body.hpp>
  #include <boost/beast/http/verb.hpp>
  #include <boost/beast/http/write.hpp>
  #include <cstdint>
  #include <cstring>
  #include <exception>
  #include <filesystem>
  #include <fstream>
  #include <ios>
  #include <iostream>
  #include <limits>
  #include <nlohmann/json.hpp>
  #include <optional>
  #include <ostream>
  #include <string>
  #include <string_view>
  #include <taglib/fileref.h>
  #include <taglib/flacfile.h>
  #include <taglib/tag.h>
  #include <taglib/tbytevector.h>

  // file format:
  // +----------------------------------+
  // |           magic header           | 8 bytes
  // +----------------------------------+
  // |               gap 1              | 2 bytes
  // +----------------------------------+
  // |     encrypted-aes-key length     | 4 bytes
  // +----------------------------------+
  // |        encrypted-aes-key         |
  // +----------------------------------+
  // |    encrypted-meta-data length    | 4 bytes
  // +----------------------------------+
  // |       encrypted-meta-data        |
  // +----------------------------------+
  // |               gap 2              | 5 bytes
  // +----------------------------------+
  // |        cover data length         | 4 bytes
  // +----------------------------------+
  // |         cover crc code           | 4 bytes
  // +----------------------------------+
  // |            cover data            |
  // +----------------------------------+
  // |       encrypted-audio-data       |
  // +----------------------------------+

  // which stands for the length of 'neteasecloudmusic'.
  constexpr size_t kLenNcmStr = 17;
  // which stands for the length of '163 Key(Don't modify):'
  constexpr size_t kLenMetaPrefix = 22;
  // which stands for the length of 'music:'
  constexpr size_t kLenJsonPrefix = 6;
  constexpr size_t kLenMetaLenBuf = 4;
  constexpr size_t kLenImageLenBuf = 4;
  constexpr size_t kLenAudioChunk = 8 * 1024;
  constexpr size_t kRc4SBoxSize = 256;
  constexpr size_t kLenCoverCrc = 4;
  constexpr size_t kLenEncAesKeyLen = 4;
  constexpr size_t kLenMagicHeader = 8;
  constexpr size_t kLenGap1 = 2;
  constexpr size_t kLenGap2 = 5;
  // field name in Json.
  constexpr const char *kJsonFieldMusicName = "musicName";
  constexpr const char *kJsonFieldArtist = "artist";
  constexpr const char *kJsonFieldFormat = "format";
  constexpr const char *kJsonFieldFormatFlac = "flac";
  constexpr const char *kJsonFieldFormatMp3 = "mp3";
  constexpr const char *kJsonFieldAlbum = "album";
  constexpr const char *kJsonFieldAlbumPic = "albumPic";
  constexpr const char *kJsonFieldAlias = "alias";
  // aes key.
  constexpr unsigned char kCoreAesKey[128] = {0x68, 0x7A, 0x48, 0x52, 0x41, 0x6D,
                                              0x73, 0x6F, 0x35, 0x6B, 0x49, 0x6E,
                                              0x62, 0x61, 0x78, 0x57};
  constexpr unsigned char kMetaAesKey[128] = {0x23, 0x31, 0x34, 0x6C, 0x6A, 0x6B,
                                              0x5F, 0x21, 0x5C, 0x5D, 0x26, 0x30,
                                              0x55, 0x3C, 0x27, 0x28};
  // taglib constants.
  // ref: https://taglib.org/api/.
  constexpr const char *kTagLibPicture = "PICTURE";
  constexpr const char *kTagLibData = "data";
  constexpr const char *kTagLibPictureType = "pictureType";
  constexpr const char *kTagLibPictureTypeFrontCover = "Front Cover";
  constexpr const char *kTagLibMimeType = "mimeType";
  constexpr const char *kTagLibMimeTypeJpeg = "image/jpeg";
  // http.
  constexpr const char *kDftCoverServerPort = "80";
  constexpr int kDftHttpVersion = 11;
  constexpr const char *kUserAgent =
      "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like "
      "Gecko) Chrome/114.0.5735.199 Safari/537.36";
  // misc.
  constexpr const char *kUsage =
      "usage: crack <audio_file_path>/<audio_files_dir> [--online]\n";
  constexpr const char *kNcmSuffix = ".ncm";
  constexpr const char *kOptionOnline = "--online";
  constexpr const char *kOutputDir = "./ncmcrack_output";

  struct AudioFileInfo {
    std::string full_path;
    std::string cover_url;
    std::string album_name;
  };

  std::optional<AudioFileInfo> Crack(const std::string &path_input_file,
                                    const std::string &path_output_folder) {
    std::ifstream encrypted_file(path_input_file, std::ios::binary);
    if (!encrypted_file) {
      std::cerr << "failed to open file: " << path_input_file << "\n";
      return std::nullopt;
    }

    encrypted_file.ignore(kLenMagicHeader + kLenGap1);

    // get encrypted aes key length.
    unsigned char key_len_buf[kLenEncAesKeyLen];
    encrypted_file.read(reinterpret_cast<char *>(key_len_buf), kLenEncAesKeyLen);
    int len_audio_key_aes_encrypted = static_cast<int>(*key_len_buf);

    // get encrypted audio aes key.
    std::unique_ptr<unsigned char[]> audio_key_aes_encrypted(
        new unsigned char[len_audio_key_aes_encrypted]);
    encrypted_file.read(reinterpret_cast<char *>(audio_key_aes_encrypted.get()),
                        len_audio_key_aes_encrypted);
    for (size_t i = 0; i < len_audio_key_aes_encrypted; ++i) {
      audio_key_aes_encrypted.get()[i] ^= 0x64;
    }

    // get the length of the meta, extract these 4 bytes in little endian.
    unsigned char meta_len_buf[kLenMetaLenBuf];
    encrypted_file.read(reinterpret_cast<char *>(meta_len_buf), kLenMetaLenBuf);
    int len_meta = *reinterpret_cast<int *>(meta_len_buf);

    // read and crack meta.
    std::unique_ptr<unsigned char[]> meta(new unsigned char[len_meta]);
    encrypted_file.read(reinterpret_cast<char *>(meta.get()), len_meta);
    for (size_t i = 0; i < len_meta; ++i) {
      meta.get()[i] ^= 0x63;
    }

    // base 64.
    len_meta -= kLenMetaPrefix;
    int max_len_meta_b64_decoded = len_meta * 3 / 4;
    std::unique_ptr<unsigned char[]> meta_b64_decoded(
        new unsigned char[max_len_meta_b64_decoded]);
    int len_b64_bytes_decoded = 0;
    if (!Base64Decode(meta.get() + kLenMetaPrefix, len_meta,
                      meta_b64_decoded.get(), max_len_meta_b64_decoded,
                      &len_b64_bytes_decoded)) {
      std::cerr << "failed to decode b64 meta\n";
      return std::nullopt;
    }

    // aes-ecb-128.
    int len_meta_bytes_decrypted = 0;
    std::unique_ptr<unsigned char[]> meta_aes_decrypted(
        new unsigned char[len_b64_bytes_decoded]);
    if (!AesEcb128Decrypt(meta_b64_decoded.get(), len_b64_bytes_decoded,
                          kMetaAesKey, meta_aes_decrypted.get(),
                          &len_meta_bytes_decrypted)) {
      std::cerr << "failed to decrypt aes meta\n";
      return std::nullopt;
    }

    encrypted_file.ignore(kLenGap2);

    // get the image size.
    std::unique_ptr<unsigned char[]> image_len_buf(
        new unsigned char[kLenImageLenBuf]);
    encrypted_file.read(reinterpret_cast<char *>(image_len_buf.get()),
                        kLenImageLenBuf);

    // maybe image crc code.
    encrypted_file.ignore(kLenCoverCrc);

    encrypted_file.ignore(*reinterpret_cast<int *>(image_len_buf.get()));

    // decode audio data with rc4.
    // decode the encrypted aes key.
    std::unique_ptr<unsigned char[]> audio_key_buf(
        new unsigned char[len_audio_key_aes_encrypted]);
    int len_audio_key_bytes_decrypted = 0;
    unsigned char *audio_key = audio_key_buf.get();
    if (!AesEcb128Decrypt(audio_key_aes_encrypted.get(),
                          len_audio_key_aes_encrypted, kCoreAesKey, audio_key,
                          &len_audio_key_bytes_decrypted)) {
      std::cerr << "failed to decrypt audio key\n";
      return std::nullopt;
    }
    audio_key = audio_key + kLenNcmStr;

    // init rc4 sbox.
    unsigned char s_box[kRc4SBoxSize];
    Rc4KeySchedule(audio_key, len_audio_key_bytes_decrypted - kLenNcmStr, s_box);

    // generate file name.
    auto meta_json = nlohmann::json::parse(std::string_view(
        reinterpret_cast<char *>(meta_aes_decrypted.get()) + kLenJsonPrefix,
        len_meta_bytes_decrypted - kLenJsonPrefix));
    auto music_format = meta_json[kJsonFieldFormat].get<std::string>();
    auto artists = meta_json[kJsonFieldArtist];
    std::string str_artists;
    for (size_t i = 0; i < artists.size(); ++i) {
      if (i) {
        str_artists.append(",");
      }
      str_artists.append(artists[i][0].get<std::string>());
    }
    auto path_audio_file =
        std::filesystem::path(path_output_folder) /
        Utf8ToSysEncoding(str_artists + " - " +
                          meta_json[kJsonFieldMusicName].get<std::string>() +
                          "." + music_format);

    // write audio data to local file.
    std::ofstream audio_file(path_audio_file,
                            std::ios::out | std::ios::trunc | std::ios::binary);
    if (!audio_file) {
      std::cerr << "failed to create file: "
                << "\n";
      return std::nullopt;
    }
    std::unique_ptr<unsigned char[]> audio_buf(new unsigned char[kLenAudioChunk]);
    while (encrypted_file) {
      encrypted_file.read(reinterpret_cast<char *>(audio_buf.get()),
                          kLenAudioChunk);
      long bytes_read = encrypted_file.gcount();
      if (bytes_read <= 0) {
        std::cerr << "failed to read audio chunks\n";
        return std::nullopt;
      }

      // rc4 decrypt.
      Rc4CustomizedDecrypt(audio_buf.get(), bytes_read, s_box);

      if (!audio_file.write(reinterpret_cast<char *>(audio_buf.get()),
                            bytes_read)) {
        std::cerr << "failed to write audio data to file\n";
        return std::nullopt;
      }
    }

  #if defined(_WIN32) || defined(_WIN64)
    AudioFileInfo audio_info{path_audio_file.string(),
                            meta_json[kJsonFieldAlbumPic].get<std::string>(),
                            meta_json[kJsonFieldAlbum].get<std::string>()};
    return std::optional<AudioFileInfo>(audio_info);
  #else
    return std::optional<AudioFileInfo>(
        AudioFileInfo{path_audio_file.string(),
                      meta_json[kJsonFieldAlbumPic].get<std::string>(),
                      meta_json[kJsonFieldAlbum].get<std::string>()});
  #endif
  }

  bool CrackAndDownload(const std::string &path_encrypted_file,
                        const std::string &output_folder, bool download_cover) {
    auto audio_file_info_opt = Crack(path_encrypted_file, output_folder);
    if (!audio_file_info_opt) {
      std::cerr << "failed to crack " << path_encrypted_file << "\n";
      return false;
    }
    if (!download_cover) {
      return true;
    }
    std::string host;
    std::string target;
    SplitUrl(audio_file_info_opt->cover_url, host, target);
    // download。
    using namespace boost;
    beast::http::response<beast::http::string_body> res;
    try {
      beast::flat_buffer buffer;
      // init connection.
      asio::io_context ioc;
      asio::ip::tcp::resolver resolver(ioc);
      beast::tcp_stream stream(ioc);
      stream.connect(resolver.resolve(host, kDftCoverServerPort));
      // assemble and send request.
      beast::http::request<beast::http::string_body> req{beast::http::verb::get,
                                                        target, kDftHttpVersion};
      req.set(beast::http::field::host, host);
      req.set(beast::http::field::user_agent, kUserAgent);
      beast::http::write(stream, req);
      // read from peer.
      beast::http::response_parser<beast::http::string_body> parser;
      parser.body_limit(std::numeric_limits<std::uint64_t>::max());
      beast::http::read(stream, buffer, parser);
      res = std::move(parser.get());
      if (res.result() != beast::http::status::ok) {
        std::cerr << "http status error " << res.result() << "\n";
        return false;
      }
    } catch (std::exception &e) {
      std::cerr << "failed to download image from "
                << audio_file_info_opt->cover_url << "\n";
      std::cerr << e.what() << "\n";
      return false;
    }
    auto body = res.body();
    // attach cover to audio file.
    TagLib::FileRef file_ref(audio_file_info_opt->full_path.c_str());
    if (file_ref.isNull() || !file_ref.tag()) {
      std::cerr << "failed to open file: " << audio_file_info_opt->full_path
                << "\n";
      return false;
    }
    file_ref.tag()->setAlbum(Utf8ToLatin1(audio_file_info_opt->album_name));
    auto byte_vec_image = TagLib::ByteVector().setData(body.data(), body.size());
    file_ref.setComplexProperties(
        kTagLibPicture, {{{kTagLibData, byte_vec_image},
                          {kTagLibPictureType, kTagLibPictureTypeFrontCover},
                          {kTagLibMimeType, kTagLibMimeTypeJpeg}}});
    if (!file_ref.save()) {
      std::cerr << "failed to save file " << audio_file_info_opt->full_path
                << "\n";
      return false;
    }
    return true;
  };

  // TODO: assume the system is little-endian for now.
  int main(const int argc, char *argv[]) {
    if (argc > 3 || argc < 2) {
      std::cerr << kUsage;
      return 1;
    }
    bool download_cover = false;
    if (argc == 3) {
      if (!strcmp(argv[2], kOptionOnline)) {
        download_cover = true;
      } else {
        std::cerr << kUsage;
        return 1;
      }
    }

    if (!std::filesystem::exists(kOutputDir)) {
      std::filesystem::create_directory(kOutputDir);
    }

    if (std::filesystem::is_directory(argv[1])) {
      bool error = false;
      for (const auto &entry : std::filesystem::directory_iterator(argv[1])) {
        if (entry.is_regular_file()) {
          const auto &fs_path_file = entry.path();
          if (auto filename = fs_path_file.filename().string();
              filename.size() > 4 &&
              filename.substr(filename.size() - 4) == kNcmSuffix) {
            error = CrackAndDownload(fs_path_file.string(), kOutputDir,
                                    download_cover);
          };
        }
      }
      return error;
    }

    return CrackAndDownload(argv[1], kOutputDir, download_cover);
  }

```

```c++
  // utils.cc
  #include <boost/locale.hpp>
  #include <boost/locale/encoding.hpp>
  #include <iostream>
  #include <openssl/evp.h>

  // return true if successful.
  bool AesEcb128Decrypt(const unsigned char *input, const int len_input,
                        const unsigned char *key, unsigned char *output,
                        int *len_bytes_decrypted) {
    // Use a unique_ptr with a custom deleter to manage EVP_CIPHER_CTX.
    std::unique_ptr<EVP_CIPHER_CTX, decltype(&EVP_CIPHER_CTX_free)> ctx(
        EVP_CIPHER_CTX_new(), EVP_CIPHER_CTX_free);
    if (!ctx) {
      std::cerr << "failed to create EVP_CIPHER_CTX.\n";
      return false;
    }
    if (EVP_DecryptInit(ctx.get(), EVP_aes_128_ecb(), key, nullptr) != 1) {
      std::cerr << "failed to initialize decryption context.\n";
      return false;
    }
    // decrypt most of the blocks.
    if (EVP_DecryptUpdate(ctx.get(), output, len_bytes_decrypted, input,
                          len_input) != 1) {
      std::cerr << "failed to update decryption.\n";
      return false;
    }
    // decrypt the last block.
    int final_len = 0;
    if (EVP_DecryptFinal_ex(ctx.get(), output + *len_bytes_decrypted,
                            &final_len) != 1) {
      std::cerr << "failed to finalize decryption. Check padding or key.\n";
      return false;
    }
    *len_bytes_decrypted += final_len;
    return true;
  }

  // return true if successful.
  bool Base64Decode(const unsigned char *input, const int input_len,
                    unsigned char *output, const int max_len_output,
                    int *len_bytes_decoded) {
    BIO *bio = BIO_new_mem_buf(input, input_len);
    BIO *base64 = BIO_new(BIO_f_base64());
    bio = BIO_push(base64, bio);
    BIO_set_flags(bio, BIO_FLAGS_BASE64_NO_NL);
    *len_bytes_decoded = BIO_read(bio, output, max_len_output);
    BIO_free_all(bio);
    return *len_bytes_decoded >= 0;
  }

  void Rc4KeySchedule(const unsigned char *key, const size_t key_len,
                      unsigned char *s_box) {
    for (int i = 0; i < 256; ++i) {
      s_box[i] = static_cast<unsigned char>(i);
    }
    int j = 0;
    for (int i = 0; i < 256; ++i) {
      j = (j + s_box[i] + key[i % key_len]) % 256;
      unsigned char temp = s_box[i];
      s_box[i] = s_box[j];
      s_box[j] = temp;
    }
  }

  // It should be noted that this is not a standard rc4 process.
  void Rc4CustomizedDecrypt(unsigned char *data, const long data_len,
                            const unsigned char *key_box) {
    for (int k = 1; k <= data_len; ++k) {
      unsigned char j = k & 0xff;
      data[k - 1] ^=
          key_box[(key_box[j] + key_box[(key_box[j] + j) & 0xff]) & 0xff];
    }
  }

  #if defined(_WIN32) || defined(_WIN64)
  #include <windows.h>
  #endif
  std::string Utf8ToSysEncoding(std::string utf8_bytes) {
  #if defined(_WIN32) || defined(_WIN64)
    LCID lcid = GetSystemDefaultLCID();
    switch (lcid) {
    case 0x0804:
      return boost::locale::conv::from_utf<char>(utf8_bytes, "GBK");
    default:
      break;
    }
  #endif
    // do nothing in both apple and linux.
    return utf8_bytes;
  }

  std::string Utf8ToLatin1(const std::string &gbk_bytes) {
    return boost::locale::conv::from_utf<char>(gbk_bytes, "ISO-8859-1");
  }

  void SplitUrl(const std::string &url, std::string &host, std::string &target) {
    size_t protocol_pos = url.find("://");
    if (protocol_pos != std::string::npos) {
      protocol_pos += 3;
    } else {
      protocol_pos = 0;
    }
    size_t path_pos = url.find('/', protocol_pos);

    if (path_pos != std::string::npos) {
      host = url.substr(protocol_pos, path_pos - protocol_pos);
      target = url.substr(path_pos);
    } else {
      host = url.substr(protocol_pos);
      target = "/";
    }
  }

```