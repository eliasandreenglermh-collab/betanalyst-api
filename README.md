import http from "node:http";
import fs from "node:fs";
import path from "node:path";
import { fileURLToPath } from "node:url";

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const PORT = process.env.PORT || 3000;
const GEMINI_API_KEY = process.env.GEMINI_API_KEY || "";
const API_FOOTBALL_KEY = process.env.API_FOOTBALL_KEY || "";
const API_SECRET = process.env.API_SECRET || "";
const ALLOWED_ORIGINS_RAW = (process.env.ALLOWED_ORIGINS || "*").split(",").map(o => o.trim());
const ALLOWED_ORIGINS = ALLOWED_ORIGINS_RAW.map(o => {
  if (o === "*") return "*";
  if (o.startsWith("*.")) return new RegExp("^https://.+\\." + o.slice(2).replace(/\./g, "\\.") + "$");
  retu
